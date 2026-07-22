# 🛒 BimBam Buy - Agente Inteligente de Atención al Cliente y Base de Conocimientos RAG

> **Proyecto Challenge Oracle ONE & Alura - Especialización en Inteligencia Artificial / Automatización**

**BimBam Buy** es una plataforma de e-commerce multiplataforma enfocada en ofrecer una experiencia de compra digital ágil, transparente y segura. Este repositorio contiene la infraestructura completa para desplegar un **Agente Virtual RAG (Retrieval-Augmented Generation)** autónomo, auto-hospedado y 100% privado, diseñado para resolver consultas de los usuarios finales sobre políticas de reembolso, programas de afiliados, tiempos/costos de envío, garantías y métodos de pago de la compañía.

---

## 📌 Tabla de Contenidos
1. [Descripción General](#-descripción-general)
2. [Arquitectura de la Solución](#-arquitectura-de-la-solución)
3. [Tecnologías y Herramientas Utilizadas](#-tecnologías-y-herramientas-utilizadas)
4. [Estructura del Proyecto](#-estructura-del-proyecto)
5. [Instrucciones para Ejecutar el Proyecto](#-instrucciones-para-ejecutar-el-proyecto)
   - [Configuración de la Base de Datos (PostgreSQL + pgvector)](#3-inicialización-de-la-base-de-datos)
6. [Ejemplos de Uso del Agente](#-ejemplos-de-uso-del-agente)
   - [Preguntas Frecuentes de la Base de Conocimiento](#preguntas-que-el-agente-puede-responder)
   - [Ejemplos de Respuestas Generadas](#ejemplos-de-respuestas-generadas-por-el-agente)
7. [Demostración en Video](#-demostración-en-video)
8. [Recomendaciones y Buenas Prácticas](#-recomendaciones-y-buenas-prácticas)

---

## 🛍️ Descripción General

BimBam Buy orienta su modelo de negocio hacia la satisfacción integral del cliente mediante políticas claras de posventa y logística optimizada. Para atender el alto volumen de consultas manteniendo respuestas fidedignas y sin alucinaciones, la solución integra un agente de inteligencia artificial que procesa de manera dinámica la documentación oficial del negocio:

* 📄 **Política de Reembolsos y Devoluciones de BimBam Buy.pdf**
* 🤝 **Programa de Afiliados de BimBam Buy.pdf**
* 🚚 **Guía de Tiempos y Costos de Envío de BimBam Buy.pdf**
* 💳 **Preguntas Frecuentes sobre Métodos de Pago de BimBam Buy.pdf**
* 🛡️ **Manual de Garantía de Productos de BimBam Buy.pdf**

El agente permite la ingesta interactiva de archivos PDF desde un Widget Web personalizado, parsea los documentos en fragmentos vectores (*vector embeddings*), almacena la información de manera semántica y responde a través de un modelo de lenguaje local (*Llama 3.1:8b*) citando explícitamente las fuentes de origen.

---

## 🏗️ Arquitectura de la Solución

La infraestructura está totalmente contenerizada mediante **Docker Compose**, lo que garantiza que el procesamiento de datos y la inferencia del modelo LLM se ejecuten localmente de forma privada y eficiente.

```
                  +-------------------------------------------------------+
                  |                  Nginx (Web Server)                   |
                  |            Interfaz Web + Chat Widget (8081)           |
                  +---------------------------+---------------------------+
                                              |
                                              v (Webhooks / REST API)
                                  +-----------------------+
                                  |     ngrok Tunnel      | (Acceso Externo / Dominio Seguro)
                                  +-----------+-----------+
                                              |
                                              v
+-----------------------------------------------------------------------------------+
|                                STACK DE DOCKER                                    |
|                                                                                   |
|   +-----------------------+        +-----------------------+                      |
|   |   Open WebUI (3000)   |------->|   Ollama Local        |                      |
|   | (Panel de prueba IA)  |        | (LLM + Embeddings)    |                      |
|   +-----------------------+        +-----------+-----------+                      |
|                                                |                                  |
|                                                v                                  |
|                                    +-----------------------+                      |
|                                    |   n8n Workflow Engine |                      |
|                                    |      (Port 5678)      |                      |
|                                    +-----------+-----------+                      |
|                                                |                                  |
|                                                v                                  |
|                                    +-----------------------+                      |
|                                    | Postgres + pgvector   |                      |
|                                    |      (Port 5432)      |                      |
|                                    +-----------+-----------+                      |
|                                                ^                                  |
|                                                |                                  |
|                                    +-----------+-----------+                      |
|                                    |   Adminer / pgAdmin   |                      |
|                                    | (Gestión BD: 5052/5050)|                      |
|                                    +-----------------------+                      |
+-----------------------------------------------------------------------------------+
```

### Flujo de Datos del Workflow RAG (n8n)
1. **Entrada de Usuario:** El usuario interactúa con la interfaz web y sube un PDF o envía una consulta.
2. **Procesamiento e Ingesta:**
   * Si es un **PDF**: Se extrae el texto (`Extract From File`), se divide en fragmentos con solapamiento (`RecursiveCharacterTextSplitter`), se generan embeddings semánticos con `nomic-embed-text` (768 dimensiones) vía Ollama y se persisten en la tabla `documents` de **PostgreSQL con pgvector**.
   * Si es una **Pregunta**: El **Agente RAG** evalúa la intención y ejecuta herramientas (*Tools*) como `buscar_documentos` (búsqueda por similitud vectorial del query) o `Listar archivos subidos` (consulta SQL directa).
3. **Generación de Respuesta:** El modelo `llama3.1:8b` redacta una respuesta precisa basada únicamente en los fragmentos contextuales extraídos, citando el archivo PDF origen.

---

## 🛠️ Tecnologías y Herramientas Utilizadas

* **Orquestador de Automatización:** [n8n](https://n8n.io/) (Flujos LangChain / Chat Trigger / Nodos Vector Store).
* **Motor de IA y Modelos Locales:** [Ollama](https://ollama.com/) (Ejecutando `llama3.1:8b` para razonamiento/síntesis y `nomic-embed-text` para generación de vectores).
* **Base de Datos Vectorial:** [PostgreSQL 16](https://www.postgresql.org/) con la extensión [pgvector](https://github.com/pgvector/pgvector) y esquema optimizado (HNSW/GIN).
* **Servidor Web Frontend:** [Nginx Alpine](https://nginx.org/) sirviendo la interfaz de chat en HTML5/CSS3 con componentes modulados de `@n8n/chat`.
* **Túnel de Conexión Pública:** [ngrok](https://ngrok.com/) para gestión de webhooks, dominios estáticos y SSL.
* **Herramientas de Administración:** [Adminer](https://www.adminer.org/) y [pgAdmin 4](https://www.pgadmin.org/) para monitoreo de la base de datos.
* **Interfaz Alternativa de Pruebas:** [Open WebUI](https://github.com/open-webui/open-webui) integrada directamente con Ollama.

---

## 📁 Estructura del Proyecto

```bash
bimbam-buy-rag/
├── docker-compose.yml              # Configuración de contenedores y servicios
├── .env.example                    # Variables de entorno de muestra
├── init.sql                        # Esquema e índices SQL para PostgreSQL con pgvector
├── web_data/
│   └── index.html                  # Interfaz principal de atención al cliente
├── workflows/
│   ├── chat_unificado_rag.json     # Workflow principal de n8n (Chat + Ingesta RAG)
│   └── api_listar_archivos.json    # Workflow API para consulta de PDFs subidos
└── docs/                           # Documentación oficial en PDF de BimBam Buy
    ├── Política de Reembolsos y Devoluciones de BimBam Buy.pdf
    ├── Programa de Afiliados de BimBam Buy.pdf
    ├── Guía de Tiempos y Costos de Envío de BimBam Buy.pdf
    ├── Preguntas_Frecuentes_sobre_Métodos_de_Pago_de_BimBam_Buy.pdf
    └── Manual de Garantía de Productos de BimBam Buy.pdf
```

---

## 🚀 Instrucciones para Ejecutar el Proyecto

### 1. Prerrequisitos
* **Docker** y **Docker Compose** instalados en el sistema.
* **ngrok** (Cuenta y Token Autenticado).
* Al menos 8 GB de memoria RAM libre para la inferencia fluida de `llama3.1:8b`.

### 2. Configuración de Variables de Entorno
Copia el archivo de variables de entorno y completa tus credenciales:

```bash
cp .env.example .env
```

Edita el archivo `.env` configurando los identificadores de tu usuario de Linux (`USER_UID` y `USER_GID`), las credenciales de PostgreSQL y tu token/dominio de ngrok:

```env
USER_UID=1000
USER_GID=1000

POSTGRES_USER=n8n_admin
POSTGRES_PASSWORD=TuPasswordSeguro123
POSTGRES_DB=n8n_data

NGROK_DOMAIN=tu-dominio-custom.ngrok-free.dev
NGROK_AUTHTOKEN=tu_authtoken_de_ngrok

PGADMIN_DEFAULT_EMAIL=admin@bimbambuy.com
PGADMIN_DEFAULT_PASSWORD=TuPasswordAdmin
```

### 3. Inicialización de la Base de Datos
Para crear las tablas de almacenamiento de vectores e índices necesarios, ejecuta el script `init.sql` incluido en el proyecto directamente en el contenedor de PostgreSQL:

```sql
-- 1. Habilitar la extensión pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- 2. Tabla principal para RAG (Postgres PGVector Store)
CREATE TABLE IF NOT EXISTS documents (
    id          BIGSERIAL PRIMARY KEY,
    content     TEXT NOT NULL,
    metadata    JSONB DEFAULT '{}'::jsonb,
    embedding   VECTOR(768) NOT NULL, -- 768 dimensiones para nomic-embed-text
    created_at  TIMESTAMPTZ DEFAULT now()
);

-- 3. Índices de optimización
CREATE INDEX IF NOT EXISTS idx_documents_metadata ON documents USING gin (metadata);
CREATE INDEX IF NOT EXISTS idx_documents_embedding_hnsw ON documents USING hnsw (embedding vector_cosine_ops);

-- 4. Tabla opcional de trazabilidad
CREATE TABLE IF NOT EXISTS uploaded_files (
    id           BIGSERIAL PRIMARY KEY,
    file_name    TEXT NOT NULL,
    session_id   TEXT,
    chunks_count INTEGER DEFAULT 0,
    uploaded_at  TIMESTAMPTZ DEFAULT now()
);
```

Puedes ejecutar este archivo en tu base de datos mediante Docker:
```bash
docker exec -i postgres-rag psql -U n8n_admin -d n8n_data < init.sql
```

### 4. Descarga de Modelos en Ollama
Inicia los servicios con Docker Compose:

```bash
docker compose up -d
```

Descarga los modelos de lenguaje y embeddings necesarios dentro del contenedor de Ollama:

```bash
docker exec -it ollama-local ollama pull llama3.1:8b
docker exec -it ollama-local ollama pull nomic-embed-text
```

### 5. Importación de Workflows en n8n
1. Ingresa a n8n mediante `http://localhost:5678`.
2. Ve a **Workflows > Import from File** e importa los archivos de la carpeta `workflows/`:
   * `chat_unificado_rag.json`
   * `api_listar_archivos.json`
3. Configura las credenciales de **PostgreSQL** y **Ollama** dentro de los nodos correspondientes.
4. **Activa** (*Activate*) ambos workflows en n8n para habilitar las URLs de producción de los Webhooks.

### 6. Configuración de la Interfaz Web
Abre el archivo `web_data/index.html` y actualiza los endpoints de los webhooks con la URL asignada por n8n:

```javascript
const CHAT_WEBHOOK_URL = 'https://tu-dominio-custom.ngrok-free.dev/webhook/ID_WEBHOOK_CHAT';
const FILES_WEBHOOK_URL = 'https://tu-dominio-custom.ngrok-free.dev/webhook/list-files';
```

Accede a la tienda y chat interactivo en `http://localhost:8081`.

---

## 💬 Ejemplos de Uso del Agente

### Preguntas que el agente puede responder:
1. **Reembolsos y Devoluciones:**
   * *"¿Cuál es el plazo máximo para solicitar la devolución de un producto defectuoso?"*
   * *"¿Quién asume los costos de envío durante un proceso de reembolso?"*
2. **Programa de Afiliados:**
   * *"¿Qué porcentaje de comisión otorga el programa de afiliados de BimBam Buy y cuál es el monto mínimo de cobro?"*
   * *"¿Cómo me inscribo al programa de afiliados?"*
3. **Logística y Envíos:**
   * *"¿Cuáles son los tiempos de entrega estándar y exprés para ciudades principales?"*
   * *"¿Tienen cobertura de envíos internacionales?"*
4. **Métodos de Pago:**
   * *"¿Cuáles son los medios de pago aceptados y cómo se procesan las cuotas con tarjeta de crédito?"*
   * *"¿Puedo pagar contra entrega?"*
5. **Garantías de Productos:**
   * *"¿Qué cubre la garantía de los productos electrónicos y qué situaciones la anulan?"*

---

### Ejemplos de Respuestas Generadas por el Agente

#### Ejemplo 1: Consulta sobre Tiempos de Envío
> **Usuario:** *"¿Cuánto tarda en llegar un pedido a Bogotá y cuál es el costo?"*
> 
> **Agente RAG:**
> *"De acuerdo con la **Guía de Tiempos y Costos de Envío de BimBam Buy.pdf**, para ciudades principales como Bogotá, los tiempos y costos son los siguientes:
> * **Envío Estándar:** Entre 2 y 4 días hábiles, con un valor de $9,000 COP (Gratis en compras superiores a $150,000 COP).
> * **Envío Exprés:** Entre 24 y 48 horas hábiles, con un costo fijo de $15,000 COP.
> Los pedidos realizados en fines de semana o festivos se procesan a partir del siguiente día hábil."*

#### Ejemplo 2: Consulta sobre Políticas de Reembolso
> **Usuario:** *"¿Si me arrepiento de una compra, cuántos días tengo para devolver el producto?"*
> 
> **Agente RAG:**
> *"Según la **Política de Reembolsos y Devoluciones de BimBam Buy.pdf**, cuentas con un plazo de **5 días hábiles** a partir de la recepción del producto para ejercer tu derecho de retracto. El producto debe encontrarse sin uso, con sus sellos de seguridad intactos y en el empaque original."*

---

## 🎥 Demostración en Video

Demostración del funcionamiento completo (subida de documentos PDF en tiempo real, sincronización en pgvector e interacción fluida con el agente virtual):

📺 **[Ver Video de Demostración del Proyecto en YouTube](https://youtu.be/q11py5NVSoA?si=NnJ85eaDIzq508v_)**

---

## 💡 Recomendaciones y Buenas Prácticas

1. **Ajuste del Tamaño de Fragmentos (*Chunk Size*):** Para documentos normativos o legales como los de BimBam Buy, se recomienda un `chunkSize` de 800 a 1000 caracteres con un `chunkOverlap` de 150 a 200 para evitar perder el contexto entre párrafos contiguos.
2. **Índices HNSW en pgvector:** El esquema incluye la creación del índice `HNSW` (*Hierarchical Navigable Small World*) sobre la columna de embeddings (`idx_documents_embedding_hnsw`), lo que acelera exponencialmente las búsquedas vectoriales por distancia coseno.
3. **Persistencia y Permisos en Linux:** Asegúrate de asignar los permisos correctos en el host local a la carpeta de montaje para que los contenedores de n8n y Postgres puedan escribir sin restricciones (`chown -R 1000:1000 ./n8n_data ./postgres_data`).
4. **Respaldo del Modelo Local:** Mantén actualizado el contenedor de Ollama y utiliza variables de temperatura bajas (`temperature: 0.1` a `0.2`) en el nodo `lmChatOllama` para asegurar respuestas hiper-enfocadas en el contexto entregado y mitigar cualquier riesgo de alucinación.

---
*Desarrollado con ❤️ para el Challenge Oracle ONE & Alura Latam.*
