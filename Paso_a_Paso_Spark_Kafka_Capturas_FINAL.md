# 🚀 Implementacion aplicacion Spark Streaming + Kafka + Hadoop

Este instructivo describe detalladamente la implementación del procesamiento de datos en tiempo real con Apache Spark, Kafka, y Hadoop, utilizando como ejemplo el conjunto de datos de capturas de la Policía Nacional de Colombia.

---

## 1. 🔌 Conexión y puesta en marcha de Hadoop

1. Abre **PuTTY** y conéctate a tu máquina virtual.
2. Inicia sesión:
   ```bash
   Usuario: hadoop
   Contraseña: hadoop
   ```
3. Inicia Hadoop:
   ```bash
   start-all.sh
   ```
4. Verifica en navegador: [http://192.168.1.4:9870](http://192.168.1.4:9870) 
### http://your-server-ip:9870 
---

## 2. 📂 Crear carpeta en HDFS y cargar dataset

1. Crear carpeta:
   ```bash
   hdfs dfs -mkdir /Tarea3
   ```
   Si ya teniamos la carpeta creada porque habiamos implementado el 1. Instructivo Análisis de Datos en Tiempo Real con Spark Streaming y Kafka_ nos va a salir un mensaje diciendo que la carpeta ya existe y podemos continuar
2. Descargar dataset:
   ```bash
   wget https://www.datos.gov.co/api/views/cukt-wz9m/rows.csv -O capturas_nacionales.csv
   ```
3. Subirlo a HDFS:
   ```bash
   hdfs dfs -put capturas_nacionales.csv /Tarea3/
   ```

---

## 3. 👤 Cambiar al usuario `vboxuser`

1. Abre una nueva terminal PuTTY.
2. Inicia sesión:
   ```bash
   Usuario: vboxuser
   Contraseña: bigdata
   ```

---
## 4. 🛠️ Iniciar servicios de Spark

### Terminal 1 – Spark
```bash
pyspark
```


## 5. 🛠️ Iniciar servicios de Kafka

### Terminal 2 – ZooKeeper
```bash
sudo rm -rf /tmp/zookeeper
sudo /opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
```

### Terminal 3 – Kafka Broker
```bash
sudo /opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
```

---

## 6. 📊 Análisis exploratorio (EDA) con Spark

### Terminal 3 – Crear y ejecutar script
```bash
nano capturas_eda.py
```

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

```bash
python3 capturas_eda.py
```

---

## 7. 🔄 Spark Structured Streaming Consumer

### Terminal 4 – Crear archivo
```bash
nano capturas_streaming.py
```

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

```bash
spark-submit --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.3 capturas_streaming.py
```

---

## 8. 📤 Kafka Producer

### Terminal 5 – Crear archivo
```bash
nano kafka_producer_capturas.py
```

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

```bash
python3 kafka_producer_capturas.py
```

---

## 9. 🔁 Reiniciar todo el entorno y verificar funcionamiento

### Apagar entorno

1. Cierra **todas las ventanas de PuTTY**.
2. Apaga la máquina virtual desde VirtualBox o consola.

### Encender y ejecutar desde cero

1. Enciende la máquina virtual.
2. Abre 5 terminales de Putty y haz lo siguiente, de manera individual en cada ventana te logueas con el usuario Vboxuser y la contraseña bigdata, en el siguiente orden:

| Terminal | Acción                                                                 |
|----------|------------------------------------------------------------------------|
| 1        | Iniciar Spark                                                          |
| 1        | Iniciar ZooKeeper                                                      |
| 2        | Iniciar Kafka                                                          |
| 3        | Ejecutar script `kafka_producer_capturas.py` (producer)                |
| 4        | Ejecutar script `capturas_streaming.py` (consumer)                     |

### NOTA: es importante primero ejecutar el script de productor y luego el del consumidor
---

## 10. 📈 Visualización y monitoreo

- Verifica los resultados en consola en cada terminal.
- Ingresa a `http://<IP_VM>:4040/streaming/` para ver los micro-batches (si aparece). Ejemplo Para este caso http://192.168.1.4:4040
- Verifica los registros por municipio, género, documento y grupo etario en tiempo real.
