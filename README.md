# databricks_project_pchasiquiza
This repository contains a MVP of end-to-end medallion architecture in Databricks.

## Sales Analytics Pipeline – Databricks (Medallion Architecture)

Este proyecto simula un caso real de retail tecnológico donde se necesita consolidar información de productos, clientes, órdenes y ventas para habilitar analítica de consumo entre 2023–2025.

El foco no es solo mover datos, sino asegurar calidad, trazabilidad y modelo analítico listo para negocio.

## Decisiones de diseño (lo importante)

### 1. Requisitos preliminares

Se utilizó la plataforma cloud de Azure, para ello se disponibilizó de lo siguiente:
  1. Subscripción de cuenta Azure
  2. Grupo de recursos
  3. 2 Servicios de Azure Databricks para desarrollo y producción
  4. Un Storage Data lake con diferentes containers (metastore, raw, bronze, silver, gold)
  5. Entry ID y Azure KeyVault
  6. Access connector vinculado con el storage (previamente definido el rol de Data Contributor)
  7. Repositorio de git para CI/CD

### 2. Configuraciones adicionales

#### Databricks
  1. Unity catalog configurado correctamente para la gobernanza
  2. Token habilitado con el git
  3. Credentials y External locations
  4. Vinculación del repositorio git

#### Github
  1. Token habilitado
  2. Repositorio con al menos dos ramas: dev y prod
  3. Repositorio de secretos para dev y prod

### 3. Uso de arquitectura Medallion

Se implementa separación en capas para evitar mezclar responsabilidades de tal manera que se pueda realizar una gobernanza de datos correcta manteniendo la trazabilidad:

Bronze → Contenido de información cruda
Silver → limpieza, estandarizaciónde la información + reglas de negocio mínimas
Gold → modelo analítico optimizado para consumo (estrella)

Al implementar las arquitectura medallón, evitamos que el usuario final consuma datos sucios o se tenga que recalcular lógica en dashboards

### 3.1. Arquitectura del pipeline
CSV (raw files)
   │
   ▼
BRONZE (raw + ingestion_time)
   │
   ▼
SILVER (clean + validated)
   │
   ▼
GOLD (dimensional model)
   │
   ▼
Dashboard / BI

### 3.2. Capas de la arquitectura
#### 🥉 Bronze Layer

Características:

Al tratarse de grandes volúmenes de información, la ingesta se realiza con PySpark y el schema se define manualmente

Se agrega:
Ingestion_time con el objetivo de tener una mayor trazabilidad de la carga de la información

#### 🥈 Silver Layer

Limpieza y estandarización de la información:
Eliminación de valores nulos críticos:
  - product_id, customer_id, order_id

Validaciones:
  - price > 0
  - quantity > 0
Normalización:
  - trim, lower
Cast de tipos:
  - DATE
  - DECIMAL

Adicional, con el fin de asegurar la integridad de la fact table en gold, se adiciona el siguiente condicional con el fin de evitar ventas huérfanas o inconsistencias en el análisis

sales_clean = sales_clean \
    .join(orders_clean, "order_id") \
    .join(products_clean, "product_id")

#### 🥇 Gold Layer (modelo analítico)

Se construye un modelo tipo estrella:

  Fact Table → fact_sales

A su vez, se agregan métricas listas para negocio:

  - net_amount
  - gross_amount
  - tax_amount -> iva cosiderando, no únicamente el valor neto
  - discount_pct  -> descuento por venta por default 5%
  - high_ticket_flag -> flag para indentificar ventas de alto valor

El objetivo principal de las métricas es evitar llevar datos de valor al consumidor, evitando caer en redudancias al momento de crear el dashboard y optimizar las consultas.

Dimensiones
- dim_customer
- dim_product
- dim_date
- dim_channel

## 4. 🔐 Seguridad (Unity Catalog)

Se separan roles:

👨‍💻 DESARROLLADORES
Acceso completo (Bronze, Silver, Gold)
Permiso MODIFY en tablas

👁️ Users (BI)
solo SELECT en Gold, accesos limitado a silver

Bronze queda completamente restringido.

## Despliegue completo a producción con CI/CD

<img width="764" height="462" alt="image" src="https://github.com/user-attachments/assets/985a7edd-4a67-4179-a818-4e00f946a380" />


## En el Dashboard podrán encontrar el análisis de ventas por año, mes, top productos y top clientes a lo largo del tiempo

<img width="1078" height="440" alt="image" src="https://github.com/user-attachments/assets/3ec9e506-ee82-4489-958b-e8c8ba87deff" />
