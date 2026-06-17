Arquitectura

Origen de datos

Base de datos transaccional (ventas).
Archivos CSV que llegan diariamente a un bucket de Google Cloud Storage.

Procesamiento

Cloud Composer orquesta el flujo.
Google Cloud Dataflow transforma y limpia los datos.

Destino

BigQuery para análisis y reportes.
Flujo técnico
Paso 1: Composer inicia el proceso

A las 02:00 AM, un DAG de Airflow ejecutado en Composer:

from airflow import DAG
from airflow.providers.google.cloud.operators.dataflow import DataflowStartFlexTemplateOperator

with DAG(
    dag_id="daily_sales_pipeline",
    schedule="0 2 * * *"
):

    run_dataflow = DataflowStartFlexTemplateOperator(
        task_id="run_sales_dataflow",
        project_id="mi-proyecto",
        location="us-central1"
    )

Composer se encarga de:

Programar la ejecución.
Monitorear el estado.
Reintentar si falla.
Enviar alertas.
Paso 2: Dataflow procesa los datos

Un pipeline de Dataflow desarrollado con Apache Beam:

import apache_beam as beam

with beam.Pipeline() as pipeline:

    (
        pipeline
        | "ReadCSV" >> beam.io.ReadFromText(
            "gs://ventas-bucket/*.csv",
            skip_header_lines=1
        )
        | "ParseRows" >> beam.Map(lambda x: x.split(","))
        | "FilterInvalid" >> beam.Filter(
            lambda row: float(row[3]) > 0
        )
        | "WriteBQ" >> beam.io.WriteToBigQuery(
            "analytics.sales"
        )
    )

Lo que hace:

Lee archivos CSV.
Convierte cada línea en columnas.
Elimina registros inválidos.
Carga la información en BigQuery.
Paso 3: Composer valida el resultado

Una vez terminado Dataflow:

validate_data = BigQueryCheckOperator(
    task_id="validate_sales",
    sql="""
    SELECT COUNT(*)
    FROM analytics.sales
    WHERE sale_date = CURRENT_DATE()
    """
)

Si hay registros:

Continúa el flujo.

Si no hay registros:

Genera una alerta.
Paso 4: Notificación

Composer envía un correo o mensaje de Slack:

send_notification = EmailOperator(
    task_id="notify",
    to="data-team@empresa.com",
    subject="Pipeline completado"
)

