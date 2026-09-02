# 📘 Documentación Técnica Exhaustiva: Sistema de Automatización Multicanal "Gestor"

> **Proyecto:** Gestor (Carmay Outfit)  
> **Última actualización:** 1 de Septiembre de 2026  
> **Servidores / Stack:** n8n v2.69.2, PostgreSQL 16, Playwright (Node.js), Baileys WhatsApp API, OmniRoute Proxy (`gemini/gemini-3.5-flash`), Google Sheets API, Google Drive API.  
> **Ubicación local:** `/home/carmay-dev/proyectos/Gestor/`

---

## 📋 Índice General

1. [Visión General del Sistema](#1-visión-general-del-sistema)
2. [Arquitectura e Infraestructura](#2-arquitectura-e-infraestructura)
3. [Orquestador Central: Marketing Expert (`OWmKP2Me3WJSzlWc`)](#3-orquestador-central-marketing-expert-owmkp2me3wjszlwc)
   - [3.1 Descripción y Objetivos](#31-descripción-y-objetivos)
   - [3.2 Desglose Nodo a Nodo](#32-desglose-nodo-a-nodo)
   - [3.3 Casos Tolerados y Auto-Recuperación](#33-casos-tolerados-y-auto-recuperación)
4. [Worker Facebook: Gestor Facebook (`fbPosterWorkflow`)](#4-worker-facebook-gestor-facebook-fbposterworkflow)
   - [4.1 Descripción y Arquitectura Anti-Ban](#41-descripción-y-arquitectura-anti-ban)
   - [4.2 Desglose Nodo a Nodo](#42-desglose-nodo-a-nodo)
   - [4.3 Script Playwright (`facebook_pw_poster.js`)](#43-script-playwright-facebook_pw_posterjs)
5. [Modulo Ingesta WhatsApp: Ingesta WhatsApp (`NgPFwrUQufkxh8K1`)](#5-modulo-ingesta-whatsapp-ingesta-whatsapp-ngpfwruqufkxh8k1)
6. [Worker WhatsApp: Worker WhatsApp (`yqOdEH1gMiuvUHN6`)](#6-worker-whatsapp-worker-whatsapp-yqodeh1gmiuvuhn6)
   - [6.1 Descripción y Protocolo Anti-Spam Estricto](#61-descripción-y-protocolo-anti-spam-estricto)
   - [6.2 Desglose Nodo a Nodo](#62-desglose-nodo-a-nodo)
   - [6.3 Modelo Simplificado de 3 Estados y Regla de Reposo 24h](#63-modelo-simplificado-de-3-estados-y-regla-de-reposo-24h)
7. [Worker Alternativo: Robot WhatsApp Queue Worker (`cVilQNdXYZAUH10M`)](#7-worker-alternativo-robot-whatsapp-queue-worker-cvilqndxyzauh10m)
8. [Modelo de Base de Datos PostgreSQL (`n8n_carmay_db`)](#8-modelo-de-base-de-datos-postgresql-n8n_carmay_db)
9. [Manual de Mantenimiento y Troubleshooting](#9-manual-de-mantenimiento-y-troubleshooting)

---

## 1. Visión General del Sistema

El ecosistema **Gestor** automatiza la publicación y difusión del catálogo de productos de **Carmay Outfit** (tienda de ropa y accesorios). Funciona como una red distribuida de microservicios y flujos de n8n que leen productos desde un catálogo maestro en Google Sheets, procesan imágenes con IA, toman decisiones de marketing multicanal y envían contenido visual a **Facebook** y **WhatsApp**.

### Principios Fundamentales de Diseño:
- **Regla Global de Contenido Visual:** Todos los envíos publican **únicamente la imagen del producto**. No se escribe texto secundario directo en las publicaciones porque las imágenes del catálogo ya contienen nombres, precios, tallas y datos de contacto de forma autosuficiente.
- **SDD (Spec-Driven Development):** Todos los cambios en la arquitectura se documentan mediante especificaciones técnicas antes de modificar el código o la base de datos.
- **Tolerancia Cero a Falsos Positivos:** Ninguna tarea se marca como `COMPLETADO` en la base de datos a menos que el servicio emisor (Playwright o Baileys) confirme el éxito del HTTP/Proceso.

---

## 2. Arquitectura e Infraestructura

```
                               ┌────────────────────────────────────────┐
                               │     GOOGLE SHEETS (Catálogo Maestro)   │
                               └───────────────────┬────────────────────┘
                                                   │
                                                   ▼
┌───────────────────────┐        ┌───────────────────────────────────┐
│ OmniRoute Proxy       │ ◄═════ │ Marketing Expert (Orquestador)    │
│ (gemini/3.5-flash)    │        │ Workflow: OWmKP2Me3WJSzlWc        │
└───────────────────────┘        └─────────┬─────────────────┬───────┘
                                           │                 │
                                  Inyecta en Cola     Inyecta en Cola
                                           │                 │
                                           ▼                 ▼
                               ┌───────────────────┐ ┌───────────────────┐
                               │ facebook_queue    │ │ whatsapp_queue    │
                               │ (PostgreSQL)      │ │ (PostgreSQL)      │
                               └─────────┬─────────┘ └─────────┬─────────┘
                                         │                     │
                                  Cron 45 min           Cron 60 min
                                         │                     │
                                         ▼                     ▼
                               ┌───────────────────┐ ┌───────────────────┐
                               │ Gestor Facebook   │ │ Gestor WhatsApp   │
                               │ (fbPosterWorkflow)│ │ (yqOdEH1gMiuvUHN6)│
                               └─────────┬─────────┘ └─────────┬─────────┘
                                         │                     │
                                  Playwright            Baileys API
                                         │                     │
                                         ▼                     ▼
                               ┌───────────────────┐ ┌───────────────────┐
                               │ Grupos de Facebook│ │ Grupos WhatsApp   │
                               └───────────────────┘ └───────────────────┘
```

### Puertos y Endpoints Clave:
- **n8n Production UI / Webhooks:** `https://carmayn8n.cloud` (`:5678` interno)
- **OmniRoute Proxy:** `http://172.16.1.1:20128/v1/chat/completions` (Gateway IP Docker)
- **Baileys WhatsApp API:** `http://baileys-sales-api:3000/send-file`
- **PostgreSQL Database:** Container `n8n-postgres` (Database: `n8n_carmay_db`)

---

## 3. Orquestador Central: Marketing Expert (`OWmKP2Me3WJSzlWc`)

### 3.1 Descripción y Objetivos
Es el **cerebro ejecutivo** del sistema. Evalúa de forma continua el catálogo completo, calcula la ventana de exclusión por canal, invoca a **Gemini 3.5 Flash** a través de OmniRoute para agrupar categorías y seleccionar qué producto se debe enviar a cada red social, inserta las decisiones en las colas correspondientes de PostgreSQL y actualiza el Google Sheet.

### 3.2 Desglose Nodo a Nodo

| # | Nombre del Nodo | Tipo | Función Técnica |
|---|---|---|---|
| 1 | `⏰ Temporizador (1 Hora)` | `scheduleTrigger` | Dispara el orquestador automáticamente cada 1 hora. |
| 2 | `117d88ca-1ffb-4295-b920-e76b5ab59430` | `webhookTrigger` | Permite disparo manual bajo demanda vía `https://carmayn8n.cloud/webhook/marketing-expert-trigger`. |
| 3 | `📊 Leer Catálogo Google Sheets` | `googleSheets` | Lee el rango completo del catálogo maestro en Google Sheets (Document ID: `1LbjE6NzELt5e_bi02DgLeOojJh_v2WIyuvaGu3fm3hU`). |
| 4 | `⚙️ Evaluar Estado Multicanal` | `code` (JS) | **Calcula elegibilidad (cooldown 48h):** Evalúa para cada producto si `FACEBOOK`, `WHATSAPP`, `INSTAGRAM`, `TELEGRAM` NO están en `ENVIADO` o si la `FECHA_ENVIO` supera las 48 horas (`now - FECHA_ENVIO >= 48h`). Devuelve el array `candidatos`. |
| 5 | `⚙️ Convertir a Archivo` | `convertToFile` | Convierte el objeto de candidatos a formato JSON binario. |
| 6 | `💾 Guardar contexto.json` | `readWriteFile` | Persiste `/data/tmp_post/contexto.json` en disco para respaldo de contexto. |
| 7 | `⚙️ Configurar y Registrar Modelo` | `code` (JS) | Lee `/data/tmp_post/last_model.json` y configura el modelo activo. Establecido por defecto en `gemini/gemini-3.5-flash`. |
| 8 | `⚙️ Formatear Petición Proxy` | `code` (JS) | Construye el cuerpo JSON del prompt. Define la directiva de negocio ("Selecciona 1 producto por canal elegible, prioriza rebaja/bajo stock por categoría, genera esquema JSON y establece resetNeeded en true si todo se agotó") y adjunta `response_format` con esquema estricto. |
| 9 | `🧠 Invocar Proxy Gemini` | `httpRequest` | Realiza llamada `POST` a `http://172.16.1.1:20128/v1/chat/completions` con token `Bearer sk-5383286d638b3ab0-d30aee-50037dc3` y modelo `gemini/gemini-3.5-flash`. |
| 10 | `💾 Guardar decision.json` | `code` (JS) | Escribe el contenido de respuesta del modelo en `/data/tmp_post/decision.json`. |
| 11 | `📖 Leer decision.json` | `readWriteFile` | Lee `/data/tmp_post/decision.json` del disco. |
| 12 | `⚙️ Parsear Decisión` | `code` (JS) | Parsea el string JSON devuelto por Gemini 3.5 Flash y desinfecta los `DRIVE_ID`s con regex `[^a-zA-Z0-9_-]` para eliminar cualquier residuo de texto. |
| 13 | `🔀 ¿Es un Reinicio?` | `if` | Evalúa si `$json.resetNeeded === true`. Si es `true`, desvía al nodo 14. Si es `false`, desvía a los nodos de normalización (15-18). |
| 14 | `📝 Resetear Catálogo a PENDIENTE` | `googleSheets` | **Auto-Recuperación:** Si no quedan productos elegibles en ninguna red, resetea las columnas `FACEBOOK`, `WHATSAPP`, `INSTAGRAM`, `TELEGRAM` de todas las filas en Google Sheets de vuelta a `PENDIENTE`. |
| 15 | `Normalizar Facebook` | `code` (JS) | Extrae `facebook_product` (`PRODUCTO` y `DRIVE_ID`) y desvía al sub-flujo de Facebook. |
| 16 | `Normalizar WhatsApp` | `code` (JS) | Extrae `whatsapp_product` (`PRODUCTO` y `DRIVE_ID`) y desvía al sub-flujo de WhatsApp. |
| 17 | `Normalizar Instagram` | `code` (JS) | Extrae `instagram_product` (`PRODUCTO` y `DRIVE_ID`). |
| 18 | `Normalizar Telegram` | `code` (JS) | Extrae `telegram_product` (`PRODUCTO` y `DRIVE_ID`). |
| 19 | `💻 Facebook Sub-flujo` | `executeWorkflow` | Invoca la ejecución de `fbPosterWorkflow` (inserta en `facebook_queue`). |
| 20 | `📲 WhatsApp Sub-flujo` | `executeWorkflow` | Invoca la ejecución del módulo de ingesta `Ingesta WhatsApp` (`NgPFwrUQufkxh8K1`). |
| 21 | `📸 Instagram Sub-flujo` | `executeWorkflow` | Invoca el sub-flujo de Instagram (`K9Fi6q9MLm2J0Ftp`). |
| 22 | `✈️ Telegram Sub-flujo` | `executeWorkflow` | Invoca el sub-flujo de Telegram (`exAcRDElb4vYlMMY`). |
| 23 | `Unificador de Canales` | `merge` | Agrupa los resultados de los 4 sub-flujos. |
| 24 | `📝 Actualizar Estados Canales` | `googleSheets` | **Actualización Iterativa (`executeOnce: false`):** Marca en el Google Sheet las columnas correspondientes como `ENVIADO` y escribe la marca de tiempo `FECHA_ENVIO` en hora Cuba (`America/Havana`). |

### 3.3 Casos Tolerados y Auto-Recuperación
- **Agotamiento del Catálogo:** Si todos los productos fueron enviados en las últimas 48h, Gemini 3.5 Flash responde `"resetNeeded": true`. El flujo captura esto, resetea la hoja a `PENDIENTE` y queda listo para el siguiente ciclo.
- **Caracteres Extraños en Drive IDs:** El nodo `⚙️ Parsear Decisión` limpia cualquier texto accidental antes de pasar el ID a los sub-flujos o a la base de datos PostgreSQL.

---

## 4. Worker Facebook: Gestor Facebook (`fbPosterWorkflow`)

### 4.1 Descripción y Arquitectura Anti-Ban
Lee de forma asíncrona la tabla PostgreSQL `facebook_queue` y publica en grupos de Facebook utilizando **Playwright** en modo Headless.
- **Cadencia:** Publica **1 foto cada 45 minutos** (máximo 32 publicaciones/día).
- **Humanización de Patrones:** Oculta firmas de automatización (`navigator.webdriver = undefined`, WebGL spoofing NVIDIA GTX 1660), utiliza curvas físicas de Bézier para el movimiento del mouse y simula tiempos de espera humanos entre clics.

### 4.2 Desglose Nodo a Nodo

| # | Nombre del Nodo | Tipo | Función Técnica |
|---|---|---|---|
| 1 | `Execute Workflow Trigger` | `executeWorkflowTrigger` | Recibe llamadas directas desde `Marketing Expert`. |
| 2 | `📥 Insertar en Cola Postgres` | `postgres` | **Deduplicación Estricta:** Inserta en `facebook_queue` solo si el producto/drive_id no está `PENDIENTE` o no fue `COMPLETADO` en los últimos 7 días. |
| 3 | `⏰ Temporizador Facebook (45 min)` | `scheduleTrigger` | Dispara el procesador de cola cada 45 minutos. |
| 4 | `🔍 Obtener Siguiente Pendiente` | `postgres` | Consulta: `SELECT id, producto, drive_id FROM facebook_queue WHERE estado = 'PENDIENTE' ORDER BY id LIMIT 1;`. |
| 5 | `⚙️ Parsear Fila` | `code` (JS) | Verifica si la consulta devolvió un producto válido (`found: true/false`). |
| 6 | `🔀 ¿Hay Pendientes?` | `if` | Si `found === true`, prosigue a la descarga. Si es `false`, finaliza la ejecución silenciosamente. |
| 7 | `📥 Descargar Imagen` | `googleDrive` | Descarga la foto desde Google Drive usando `drive_id`. |
| 8 | `💾 Guardar en VPS` | `readWriteFile` | Escribe la imagen temporalmente en `/data/tmp_post/facebook_item.jpg`. |
| 9 | `💻 Publicar en Facebook (Playwright)` | `executeCommand` | Ejecuta: `node /data/facebook_pw_poster.js "27546646424927431" "/data/tmp_post/facebook_item.jpg"`. Habilitado `"onError": "continueErrorOutput"`. |
| 10 | `🔀 ¿Publicación Exitosa?` | `if` | **Validación de Éxito:** Evalúa si `!$json.error && $json.exitCode === 0`. Si es verdadero desvía al nodo 11 (`COMPLETADO`). Si es falso desvía al nodo 12 (`FALLIDO`). |
| 11 | `📝 Marcar COMPLETADO` | `postgres` | Consulta: `UPDATE facebook_queue SET estado = 'COMPLETADO', fecha_envio = NOW() WHERE id = ...;`. |
| 12 | `❌ Marcar FALLIDO` | `postgres` | Consulta: `UPDATE facebook_queue SET estado = 'FALLIDO', fecha_envio = NOW() WHERE id = ...;`. |
| 13 | `🧹 Limpiar Temporal` | `executeCommand` | **Limpieza de Disco:** Ejecuta `rm -f /data/tmp_post/facebook_item.jpg /data/tmp_post/pw_step*.png 2>/dev/null || true` para mantener el uso de disco en 0 KB. |

### 4.3 Script Playwright (`facebook_pw_poster.js`)
- **Ubicación:** `/opt/n8ncarmay/facebook_pw_poster.js` (`/data/facebook_pw_poster.js` en Docker).
- **Navegador con Perfil Persistente (`launchPersistentContext`):** Almacena y reutiliza el perfil completo del navegador en `/data/tmp_post/facebook_browser_profile` (cookies, `localStorage`, `IndexedDB`), evitando que Facebook invalide la sesión.
- **Auto-Renovación de Cookies en Disco:** Al finalizar cada publicación exitosa, captura las cookies renovadas por Facebook y actualiza `/data/REDACTED_COOKIES` automáticamente.
- **Auto-Limpieza de Bloqueo (`SingletonLock`):** Borra automáticamente cualquier archivo de bloqueo `SingletonLock` en el perfil si el contenedor o el VPS sufren una caída repentina.
- **Límite Estricto de Caché en Disco:** Configurado `--disk-cache-size=104857600` (máximo 100 MB) para controlar el almacenamiento.
- **Caché de Binarios de Chromium:** Configurado `process.env.PLAYWRIGHT_BROWSERS_PATH = '/data/tmp_post/.cache'` para mantener los binarios entre reinicios del contenedor.
- **Detección Estricta de Sesión:** Si la sesión es revocada, sale con **código de error 2** (`SESIÓN INVÁLIDA`), capturado por n8n para marcar el ítem como `FALLIDO` sin congelar la cola.

---

## 5. Modulo Ingesta WhatsApp: Ingesta WhatsApp (`NgPFwrUQufkxh8K1`)

### 5.1 Descripción y Arquitectura
Es el **productor liviano** del canal de WhatsApp. Recibe sincrónicamente la solicitud desde Marketing Expert e inserta el producto en la cola PostgreSQL `whatsapp_queue`.

- **Duración típica de ejecución:** **18 ms** (0.018 segundos).
- **Nodos (2):** `Execute Workflow Trigger` → `Insert Queue`.

### 5.2 Consulta SQL de Inserción (`Insert Queue`)
```sql
INSERT INTO whatsapp_queue (producto, drive_id) 
SELECT LEFT('{{ ($json.PRODUCTO || $json.producto || '').replace(/Limpiar.*$/,'').replace(/\s+/g,' ').trim() }}', 255), 
       regexp_replace('{{ $json.DRIVE_ID || $json.drive_id }}', '\s.*$', '') 
WHERE regexp_replace('{{ $json.DRIVE_ID || $json.drive_id }}', '\\s.*$', '') ~ '^[a-zA-Z0-9_-]{25,}$' 
  AND NOT EXISTS (
    SELECT 1 FROM whatsapp_queue 
    WHERE (regexp_replace(producto, '\s+', ' ', 'g') = regexp_replace('{{ ($json.PRODUCTO || $json.producto || '').replace(/Limpiar.*$/,'').replace(/\s+/g,' ').trim() }}', '\s+', ' ', 'g') 
        OR drive_id = regexp_replace('{{ $json.DRIVE_ID || $json.drive_id }}', '\s.*$', '')) 
      AND (estado = 'PENDIENTE' OR (estado = 'COMPLETADO' AND fecha_envio > NOW() - INTERVAL '15 days'))
  ) 
  AND NOT ('{{ ($json.PRODUCTO || $json.producto || '').replace(/Limpiar.*$/,'').replace(/\s+/g,' ').trim() }}' ~ '\s*\(\d+\)\s*$');
```

- **Filtro de Variantes / Fotos Secundarias:** Descarta automáticamente nombres terminados en números entre paréntesis como `(2)` o `(3)` para no duplicar publicaciones de distintas fotos o colores del mismo producto.

---

## 6. Worker WhatsApp: Worker WhatsApp (`yqOdEH1gMiuvUHN6`)

### 6.1 Descripción y Protocolo Anti-Spam Estricto
Es el **consumidor asincrónico** del canal de WhatsApp. Procesa la cola de PostgreSQL `whatsapp_queue` y envía imágenes multipart a la API local de Baileys (`http://baileys-sales-api:3000/send-file`).

### 🛠️ Protocolo Anti-Spam de Producción:
1. **Límite Máximo Diario:** Máximo **30 fotos totales al día** en hora Cuba (`America/Havana`).
2. **Bloqueo Horario Nocturno:** **Prohibido enviar entre 12:00 AM y 07:00 AM (Hora Cuba)** (`EXTRACT(HOUR) >= 7`).
3. **Frecuencia del Temporizador:** Dispara cada **60 minutos**.
4. **Jitter Inicial:** Retraso aleatorio de **1 a 30 minutos** antes de procesar cada ciclo.
5. **Tamaño de Lote:** Máximo **2 a 3 imágenes por tanda**.
6. **Pausa Orgánica entre Fotos:** Espera de **60 a 180 segundos** (1 a 3 minutos) por cada imagen dentro del lote.

### 6.2 Desglose Nodo a Nodo

| # | Nombre del Nodo | Tipo | Función Técnica |
|---|---|---|---|
| 1 | `Schedule Trigger` | `scheduleTrigger` | Disparador de producción cada 60 minutos. |
| 2 | `Calc Initial Delay` | `code` (JS) | Genera delay aleatorio inicial de `1 a 30 minutos`. |
| 3 | `Wait Initial Delay` | `wait` | Aplica la espera del Jitter Inicial. |
| 4 | `Get Target Group` | `postgres` | Selecciona el grupo activo con la fecha de último envío más antigua (`whatsapp_groups`). |
| 5 | `Get Pending Batch` | `postgres` | **Filtro de 3 Estados y Reposo 24h:** Consulta productos `PENDIENTE` o productos `FALLIDO` con `fecha_envio <= NOW() - 24 hours` que no hayan sido enviados al grupo destino. |
| 6 | `If Has Pending` | `if` | **Validación Anti-Trash:** Verifica que el item contenga un `id` e `producto` reales (`!undefined`). |
| 7 | `Loop Batch` | `splitInBatches` | Recorre el lote de 2-3 productos ítem por ítem. |
| 8 | `Update Group Last Sent` | `postgres` | Actualiza `ultimo_envio = NOW()` en el grupo seleccionado. |
| 9 | `Calc Organic Delay` | `code` (JS) | Calcula pausa orgánica de **60 a 180 segundos**. |
| 10 | `Wait Organic Delay` | `wait` | Aplica la pausa orgánica entre imágenes. |
| 11 | `Download Image` | `httpRequest` | Descarga la imagen de Google Drive. Configurado `"onError": "continueErrorOutput"`. |
| 12 | `🔀 ¿Descarga Exitosa?` | `if` | Evalúa si `$json.error` no existe. |
| 13 | `Send WhatsApp` | `httpRequest` | Envía imagen multipart a `http://baileys-sales-api:3000/send-file`. Configurado `"onError": "continueErrorOutput"`. |
| 14 | `🔀 ¿Envío Exitoso?` | `if` | Evalúa si el envío por Baileys fue exitoso (`!$json.error`). |
| 15 | `Mark Completed` | `postgres` | Inserta en `whatsapp_sent_log` y actualiza `grupos_enviados`. Si se envió a todos los grupos activos, marca como `COMPLETADO`. |
| 16 | `Send Download Error Alert` | `httpRequest` | Envía alerta de descarga fallida al WhatsApp del admin (`REDACTED_PHONE_1`). |
| 17 | `Send Send Error Alert` | `httpRequest` | Envía alerta de envío fallido al WhatsApp del admin (`REDACTED_PHONE_1`). |
| 18 | `❌ Marcar como Fallido` | `postgres` | **Marcado Directo a FALLIDO:** Ejecuta `UPDATE whatsapp_queue SET estado = 'FALLIDO', fecha_envio = NOW() WHERE id = COALESCE(${id}, -1);`. |

### 6.3 Modelo Simplificado de 3 Estados y Regla de Reposo 24h

El sistema utiliza únicamente 3 estados:
- **`PENDIENTE`**: Producto en cola listo para envío.
- **`COMPLETADO`**: Producto enviado con éxito a todos los grupos activos habilitados.
- **`FALLIDO`**: Producto que presentó error en descarga o envío.

#### **Mecánica de Reposo 24h para Fallidos:**
Cuando un producto pasa a `FALLIDO`, se registra su `fecha_envio = NOW()`. La consulta `Get Pending Batch` lo ignora durante 24 horas exactas. Transcurrido ese plazo, el producto vuelve a ser seleccionado automáticamente para reintento únicamente en los grupos donde aún no se haya publicado.

---

## 7. Worker Alternativo: Robot WhatsApp Queue Worker (`cVilQNdXYZAUH10M`)

Este flujo opera directamente sobre Google Sheets (sin PostgreSQL) y utiliza Green API. Se mantiene activo para procesos secundarios (como envío de catálogos en formato ZIP o lotes masivos administrativos).
- **Modos:** Individual (máx 5) y Modo ZIP (lotes de hasta 30 comprimidos en `.zip`).

---

## 8. Modelo de Base de Datos PostgreSQL (`n8n_carmay_db`)

### Tabla: `facebook_queue`
```sql
CREATE TABLE facebook_queue (
    id SERIAL PRIMARY KEY,
    producto VARCHAR(255) NOT NULL,
    drive_id VARCHAR(255) NOT NULL,
    estado VARCHAR(50) DEFAULT 'PENDIENTE', -- PENDIENTE, COMPLETADO, FALLIDO
    fecha_envio TIMESTAMP
);
```

### Tabla: `whatsapp_queue`
```sql
CREATE TABLE whatsapp_queue (
    id SERIAL PRIMARY KEY,
    producto VARCHAR(255) NOT NULL,
    drive_id VARCHAR(255) NOT NULL,
    estado VARCHAR(50) DEFAULT 'PENDIENTE', -- PENDIENTE, COMPLETADO, FALLIDO
    grupos_enviados TEXT[] DEFAULT '{}',
    fecha_solicitud TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    fecha_envio TIMESTAMP
);
```

### Tabla: `whatsapp_sent_log`
```sql
CREATE TABLE whatsapp_sent_log (
    id SERIAL PRIMARY KEY,
    producto VARCHAR(255) NOT NULL,
    drive_id VARCHAR(255) NOT NULL,
    group_id VARCHAR(255) NOT NULL,
    sent_at TIMESTAMP DEFAULT NOW()
);
```

### Tabla: `whatsapp_groups`
```sql
CREATE TABLE whatsapp_groups (
    id VARCHAR(255) PRIMARY KEY,
    nombre VARCHAR(255) NOT NULL,
    activo BOOLEAN DEFAULT TRUE,
    ultimo_envio TIMESTAMP,
    orden INT
);
```

---

## 8. Manual de Mantenimiento y Troubleshooting

### A. Renovación de Cookies de Facebook
Si Playwright arroja el error `SESIÓN INVÁLIDA: Las cookies de Facebook han expirado`:
1. Abre tu navegador en la PC donde estés logueado en Facebook.
2. Exporta las cookies usando una extensión en formato JSON o Netscape.
3. Convierte e inyecta las cookies en `/opt/n8ncarmay/REDACTED_COOKIES` (`/data/REDACTED_COOKIES` en Docker).
4. Verifica la sesión ejecutando en terminal:
   ```bash
   docker exec n8n-app node /data/facebook_pw_poster.js
   ```

### B. Verificación de Conectividad OmniRoute
Si el nodo `🧠 Invocar Proxy Gemini` da error de conexión:
1. Comprueba que el proceso OmniRoute escuche en el port 20128 del VPS.
2. Prueba la conectividad desde el contenedor de n8n:
   ```bash
   docker exec n8n-app curl -s http://172.16.1.1:20128/v1/models -H "Authorization: Bearer sk-5383286d638b3ab0-d30aee-50037dc3"
   ```

### C. Purga o Reset Completo de la Cola
Si deseas reiniciar todo el sistema a cero:
```bash
docker exec n8n-postgres psql -U carmay_n8n -d n8n_carmay_db -c "TRUNCATE facebook_queue, whatsapp_queue, whatsapp_sent_log RESTART IDENTITY;"
docker exec n8n-postgres psql -U carmay_n8n -d n8n_carmay_db -c "UPDATE whatsapp_groups SET ultimo_envio = NULL;"
rm -f /opt/n8ncarmay/tmp_post/contexto.json /opt/n8ncarmay/tmp_post/decision.json /opt/n8ncarmay/tmp_post/*.png
```

---

*Documentación técnica generada y verificada en el repositorio local `/home/carmay-dev/proyectos/Gestor/DOCUMENTACION_FLUJOS_GESTOR.md`.*