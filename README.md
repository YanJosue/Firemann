# Proyecto Fireman: Propuesta de Implementación Detallada

## **Introducción**

Este documento presenta una propuesta de solución completa y detallada para la implementación del proyecto "Fireman", un Security Data Lake. El objetivo es servir como una guía secuencial para un equipo de implementación, cubriendo desde la arquitectura y el diseño del modelo de datos hasta el plan de ejecución por fases y las respuestas a preguntas clave.

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

## **Sección 2: El Esquema Canónico JSON (La Piedra Angular)**

Este es el "formulario de reporte universal". Se define UNA sola vez y se va rellenando a lo largo del proceso. Su estructura base no cambia.

### **2.1 El Esquema Canónico Final**

Este es el archivo `canonical_schema.json` que servirá como la única fuente de verdad para la estructura de nuestros datos.

```json
{
  "$schema": "[http://json-schema.org/draft-07/schema#](http://json-schema.org/draft-07/schema#)",
  "title": "Fireman Canonical Event Schema",
  "description": "El formato único y estandarizado para todos los eventos de seguridad procesados en el pipeline Fireman.",
  "type": "object",
  "required": [
    "event_id",
    "ingestion_timestamp",
    "source_system",
    "event",
    "raw_log"
  ],
  "properties": {
    "event_id": {
      "description": "Identificador Único Universal (UUID) para el evento.",
      "type": "string",
      "format": "uuid"
    },
    "ingestion_timestamp": {
      "description": "Fecha y hora en formato ISO 8601 UTC cuando el evento fue recibido.",
      "type": "string",
      "format": "date-time"
    },
    "source_system": {
      "description": "El sistema que originó el log (ej. SonicWall-TZ2070, MS-Defender).",
      "type": "string"
    },
    "event": {
      "description": "Detalles sobre la acción que ocurrió.",
      "type": "object",
      "properties": {
        "type": { "type": "string", "enum": ["network_flow", "login", "file_access", "process_creation", "email_event", "auth_event"] },
        "action": { "type": "string", "enum": ["allow", "deny", "quarantine", "delete", "login_success", "login_failure", "file_created", "file_read", "file_modified"] },
        "severity": { "type": "string", "enum": ["info", "low", "medium", "high", "critical"] }
      },
      "required": ["type"]
    },
    "source": {
      "description": "Describe el actor que inició el evento.",
      "type": "object",
      "properties": {
        "ip": { "type": "string", "format": "ipv4" },
        "port": { "type": "integer", "minimum": 0, "maximum": 65535 },
        "hostname": { "type": "string" },
        "user": {
          "type": "object",
          "properties": {
            "id": { "type": "string" },
            "name": { "type": "string" },
            "email": { "type": "string", "format": "email" }
          }
        },
        "process": {
          "type": "object",
          "properties": {
            "pid": { "type": "integer" },
            "name": { "type": "string" },
            "path": { "type": "string" }
          }
        }
      }
    },
    "target": {
      "description": "Describe el objetivo del evento.",
      "type": "object",
      "properties": {
        "ip": { "type": "string", "format": "ipv4" },
        "port": { "type": "integer", "minimum": 0, "maximum": 65535 },
        "hostname": { "type": "string" },
        "file": {
          "type": "object",
          "properties": {
            "name": { "type": "string" },
            "path": { "type": "string" },
            "hash_md5": { "type": "string" },
            "hash_sha256": { "type": "string" }
          }
        }
      }
    },
    "network": {
      "description": "Detalles específicos de eventos de red.",
      "type": "object",
      "properties": {
        "protocol": { "type": "string" },
        "bytes_in": { "type": "integer" },
        "bytes_out": { "type": "integer" },
        "direction": { "type": "string", "enum": ["inbound", "outbound", "internal"] }
      }
    },
    "email": {
      "description": "Detalles específicos de eventos de correo electrónico.",
      "type": "object",
      "properties": {
        "sender_address": { "type": "string", "format": "email" },
        "recipient_address": { "type": "string", "format": "email" },
        "subject": { "type": "string" },
        "attachment_count": { "type": "integer" }
      }
    },
    "enrichment": {
      "description": "Contenedor para toda la información añadida por los servicios de enriquecimiento.",
      "type": "object",
      "properties": {}
    },
    "raw_log": {
      "description": "El log original completo, sin procesar, para auditoría.",
      "type": "string"
    }
  }
}