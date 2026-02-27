🏗️ ARQUITECTURA DEL PIPELINE
Raw Data (CSV / S3 / ADLS)
        ↓
🟤 BRONZE  (Datos crudos - Delta)
        ↓
⚪ SILVER  (Datos limpios y validados - Delta)
        ↓
🟡 GOLD    (Datos agregados para BI - Delta)

Ejemplo dataset: ventas_2025.csv

Columnas:

id_venta, fecha, cliente, producto, categoria, cantidad, precio_unitario
🚀 PARTE 1 — PIPELINE ETL COMPLETO EN PYSPARK
🟤 1️⃣ BRONZE LAYER (Raw Ingestion)
Notebook: 01_bronze_ingestion
from pyspark.sql.functions import current_timestamp

# Leer CSV desde Data Lake
df_raw = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("/mnt/datalake/raw/ventas_2025.csv")

# Agregar metadata
df_bronze = df_raw.withColumn("ingestion_timestamp", current_timestamp())

# Guardar como Delta
df_bronze.write.format("delta") \
    .mode("overwrite") \
    .save("/mnt/datalake/bronze/ventas")

# Crear tabla
spark.sql("""
CREATE TABLE IF NOT EXISTS bronze_ventas
USING DELTA
LOCATION '/mnt/datalake/bronze/ventas'
""")
⚪ 2️⃣ SILVER LAYER (Cleaning & Transformation)
Notebook: 02_silver_transformation
from pyspark.sql.functions import col, to_date, when

df_bronze = spark.read.format("delta") \
    .load("/mnt/datalake/bronze/ventas")

df_silver = df_bronze \
    .withColumn("fecha", to_date(col("fecha"), "yyyy-MM-dd")) \
    .withColumn("cantidad", col("cantidad").cast("int")) \
    .withColumn("precio_unitario", col("precio_unitario").cast("double")) \
    .withColumn("total_venta", col("cantidad") * col("precio_unitario")) \
    .filter(col("fecha").isNotNull()) \
    .filter(col("cantidad") > 0)

df_silver.write.format("delta") \
    .mode("overwrite") \
    .save("/mnt/datalake/silver/ventas")

spark.sql("""
CREATE TABLE IF NOT EXISTS silver_ventas
USING DELTA
LOCATION '/mnt/datalake/silver/ventas'
""")
🟡 3️⃣ GOLD LAYER (Business Aggregation)
Notebook: 03_gold_aggregation
from pyspark.sql.functions import sum, month, year

df_silver = spark.read.format("delta") \
    .load("/mnt/datalake/silver/ventas")

df_gold = df_silver \
    .withColumn("year", year("fecha")) \
    .withColumn("month", month("fecha")) \
    .groupBy("year", "month", "categoria") \
    .agg(sum("total_venta").alias("ventas_totales"))

df_gold.write.format("delta") \
    .mode("overwrite") \
    .save("/mnt/datalake/gold/ventas_mensuales")

spark.sql("""
CREATE TABLE IF NOT EXISTS gold_ventas_mensuales
USING DELTA
LOCATION '/mnt/datalake/gold/ventas_mensuales'
""")
⏰ 4️⃣ ORQUESTACIÓN

En Databricks Workflows:

Crear Job:

Task 1 → 01_bronze_ingestion
Task 2 → 02_silver_transformation
Task 3 → 03_gold_aggregation

Schedule → diario / cada hora.

💡 BONUS (Para portafolio senior)

Puedes agregar:

# Upsert con MERGE (modo incremental)
spark.sql("""
MERGE INTO silver_ventas t
USING bronze_ventas s
ON t.id_venta = s.id_venta
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
""")

Eso demuestra manejo avanzado de Delta Lake.

🚀 PARTE 2 — PIPELINE COMPLETO EN DATABRICKS SQL
🟤 BRONZE (SQL)
CREATE TABLE bronze_ventas
USING DELTA
LOCATION '/mnt/datalake/bronze/ventas'
AS
SELECT *,
       current_timestamp() AS ingestion_timestamp
FROM csv.`/mnt/datalake/raw/ventas_2025.csv`
⚪ SILVER (SQL)
CREATE OR REPLACE TABLE silver_ventas
USING DELTA
AS
SELECT
    id_venta,
    TO_DATE(fecha, 'yyyy-MM-dd') AS fecha,
    cliente,
    producto,
    categoria,
    CAST(cantidad AS INT) AS cantidad,
    CAST(precio_unitario AS DOUBLE) AS precio_unitario,
    CAST(cantidad AS INT) * CAST(precio_unitario AS DOUBLE) AS total_venta
FROM bronze_ventas
WHERE cantidad > 0
AND fecha IS NOT NULL;
🟡 GOLD (SQL)
CREATE OR REPLACE TABLE gold_ventas_mensuales
USING DELTA
AS
SELECT
    YEAR(fecha) AS year,
    MONTH(fecha) AS month,
    categoria,
    SUM(total_venta) AS ventas_totales
FROM silver_ventas
GROUP BY YEAR(fecha), MONTH(fecha), categoria;
