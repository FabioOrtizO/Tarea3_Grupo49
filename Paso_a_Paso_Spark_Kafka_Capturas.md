
# Paso a Paso Definitivo - Procesamiento de Datos con Spark Streaming y Kafka

Este instructivo describe detalladamente la implementación del procesamiento de datos en tiempo real con Apache Spark, Kafka, y Hadoop, utilizando como ejemplo el conjunto de datos de capturas de la Policía Nacional de Colombia.

## 1. Conexión y puesta en marcha de Hadoop

1. Abre PuTTY y conéctate a tu máquina virtual.
2. Si aparece una alerta de seguridad, haz clic en **Aceptar**.
3. Inicia sesión como:
   ```bash
   Usuario: hadoop
   Password: hadoop
   ```
4. Inicia el clúster Hadoop:
   ```bash
   start-all.sh
   ```
5. Verifica Hadoop en: `http://192.168.1.13:9870`

## 2. Crear carpeta y cargar el dataset en HDFS

1. Crear carpeta:
   ```bash
   hdfs dfs -mkdir /Tarea3
   ```
2. Descargar dataset desde datos.gov.co:
   ```bash
   wget https://www.datos.gov.co/api/views/cukt-wz9m/rows.csv -O capturas_nacionales.csv
   ```
3. Cargar a HDFS:
   ```bash
   hdfs dfs -put capturas_nacionales.csv /Tarea3/
   hdfs dfs -ls /Tarea3
   ```

## 3. Cambiar al usuario `vboxuser`

1. Abrir nueva terminal PuTTY
2. Iniciar sesión:
   ```bash
   Usuario: vboxuser
   Password: bigdata
   ```

## 4. Iniciar servicios de Kafka

### Terminal 1 - Iniciar ZooKeeper:
```bash
sudo rm -rf /tmp/zookeeper
sudo /opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
```

### Terminal 2 - Iniciar Kafka:
```bash
sudo /opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
```

### Verificar puertos:
```bash
sudo ss -tunelp | grep 2181  # ZooKeeper
sudo ss -tunelp | grep 9092  # Kafka
```

## 5. Análisis exploratorio con Spark (EDA)

### Terminal 3 - Crear script:
```bash
nano capturas_eda.py
```

### Contenido:
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col

spark = SparkSession.builder.appName("EDA Capturas").getOrCreate()
df = spark.read.option("header", "true").csv("hdfs://localhost:9000/Tarea3/capturas_nacionales.csv")
df.printSchema()
df = df.dropDuplicates().dropna()
print("Total registros:", df.count())
df.groupBy("genero").count().show()
df.groupBy("descripcion_conducta").count().orderBy("count", ascending=False).show()
df.write.mode("overwrite").parquet("hdfs://localhost:9000/Tarea3/limpio")
```

### Ejecutar:
```bash
python3 capturas_eda.py
```

## 6. Crear y ejecutar Spark Streaming Consumer con estructura formal

### Terminal 4 - Crear archivo:
```bash
nano capturas_streaming.py
```

### Contenido:
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import from_json, col, sum as spark_sum
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

spark = SparkSession.builder.appName("CapturasStreaming").getOrCreate()
spark.sparkContext.setLogLevel("WARN")

schema = StructType([
    StructField("departamento", StringType()),
    StructField("municipio", StringType()),
    StructField("genero", StringType()),
    StructField("descripcion_conducta", StringType()),
    StructField("grupo_etario", StringType()),
    StructField("tipo_documento", StringType()),
    StructField("cantidad", StringType())
])

df = spark.readStream.format("kafka")     .option("kafka.bootstrap.servers", "localhost:9092")     .option("subscribe", "capturas_topic")     .load()

parsed_df = df.select(from_json(col("value").cast("string"), schema).alias("data")).select("data.*")
parsed_df = parsed_df.withColumn("cantidad", col("cantidad").cast(IntegerType()))

municipio_stats = parsed_df.groupBy("municipio").agg(spark_sum("cantidad").alias("total_municipio"))
documento_stats = parsed_df.groupBy("tipo_documento").agg(spark_sum("cantidad").alias("total_documento"))
genero_stats = parsed_df.groupBy("genero").agg(spark_sum("cantidad").alias("total_genero"))
grupo_stats = parsed_df.groupBy("grupo_etario").agg(spark_sum("cantidad").alias("total_grupo"))

query1 = municipio_stats.writeStream.outputMode("complete").format("console").option("truncate", False).start()
query2 = documento_stats.writeStream.outputMode("complete").format("console").option("truncate", False).start()
query3 = genero_stats.writeStream.outputMode("complete").format("console").option("truncate", False).start()
query4 = grupo_stats.writeStream.outputMode("complete").format("console").option("truncate", False).start()

query1.awaitTermination()
query2.awaitTermination()
query3.awaitTermination()
query4.awaitTermination()
```

### Ejecutar:
```bash
spark-submit --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.3 capturas_streaming.py
```

## 7. Crear y ejecutar Kafka Producer

### Terminal 5 - Crear archivo:
```bash
nano kafka_producer_capturas.py
```

### Contenido:
```python
import time, json, csv
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers='localhost:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    api_version=(0, 10)
)

with open('capturas_nacionales.csv', newline='', encoding='utf-8') as csvfile:
    reader = csv.DictReader(csvfile)
    for row in reader:
        producer.send('capturas_topic', value=row)
        print("Enviado:", row)
        time.sleep(0.5)
```

### Ejecutar:
```bash
python3 kafka_producer_capturas.py
```

## 8. Reiniciar y ejecutar el sistema completo

1. Cierra todas las ventanas de PuTTY.
2. Apaga la máquina virtual.
3. Enciéndela nuevamente.
4. Abre nuevas sesiones PuTTY y repite los pasos 4 a 7.

## 9. Visualización y monitoreo

- Verifica la consola del consumidor para ver estadísticas.
- Accede a `http://<IP_VM>:4040/streaming/` para monitorear los micro-batches.
