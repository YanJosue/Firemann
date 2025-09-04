# Proyecto Fireman: Propuesta de Implementación Detallada

## **Introducción**

Este documento presenta una propuesta de solución. 
---

## **Sección 1: Propuesta de Implementación**

### **1.1 Objetivo del Proyecto**

El objetivo principal del proyecto "Fireman" es crear una plataforma centralizada para recolectar, procesar, enriquecer y analizar todos los eventos de seguridad de la organización en tiempo real. Esto permitirá una detección de amenazas más rápida, facilitará la investigación de incidentes y proporcionará una visibilidad completa de la postura de seguridad de la infraestructura. Se utilizará la metodología ETL-E (Extract, Transform, Load - Enrich).

### **1.2 Arquitectura General**

El sistema se concibe como una línea de ensamblaje de datos de seguridad con los siguientes componentes:

| Componente | Tecnología Propuesta | Responsabilidad |
| :--- | :--- | :--- |
| **Conectores** | Python / Go | **La Puerta de Entrada:** Extraen logs de las fuentes (SonicWall, Microsoft, etc.) y los transforman a nuestro formato JSON estándar. |
| **Cola de Mensajes** | Apache Kafka | **La Banda Transportadora:** Recibe los datos de los conectores y los distribuye a los siguientes componentes de forma segura y ordenada. |
| **Servicio de Enriquecimiento** | Python (FastAPI) | **El Experto en Contexto:** Toma un evento, investiga (ej. la reputación de una IP) y le añade esa información valiosa. |
| **Cargador a Data Lake** | Python | **El Archivista a Largo Plazo:** Guarda todos los datos enriquecidos de forma económica en un lago de datos para análisis futuros. |
| **Cargador a Buscador** | Logstash | **El Indexador Rápido:** Envía los datos enriquecidos a un motor de búsqueda para consultas y visualizaciones en tiempo real. |
| **Data Lake** | AWS S3 (o similar) | **El Almacén Histórico:** Guarda todo. Es barato y perfecto para análisis masivos y entrenamiento de modelos de IA. |
| **Motor de Búsqueda** | Elasticsearch | **El Buscador Inteligente:** Permite buscar eventos específicos muy rápidamente y crear dashboards (con Kibana). |

---
