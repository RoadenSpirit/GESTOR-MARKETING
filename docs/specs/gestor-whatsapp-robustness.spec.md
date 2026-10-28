# Especificación Técnica (SDD): Robustez y Activación de Gestor WhatsApp

Este documento define los cambios técnicos para el workflow `Gestor WhatsApp` (`yqOdEH1gMiuvUHN6`) con el objetivo de reactivar el temporizador automático y hacerlo tolerante a fallos de descarga de imágenes en Google Drive.

## 1. Contexto y Problema

El workflow `Gestor WhatsApp` posee dos disparadores:
1. **Integrated (`Execute Workflow Trigger`):** Inserta ítems en `whatsapp_queue` con estado `PENDIENTE`.
2. **Cron (`Schedule Trigger`):** Procesador automático cada 30 minutos. Está actualmente **desactivado** (`"disabled": true`).

**Causa del bloqueo:** El 16 de julio de 2026, una descarga de imagen con `drive_id` inválido retornó un error HTTP 404 en el nodo `Download Image`. Debido a que este nodo no tiene control de errores, abortó la ejecución de todo el workflow, congelando el bucle de procesamiento y dejando los 19 productos restantes en la cola con estado `PENDIENTE`.

## 2. Cambios Propuestos

### 2.1. Modificación en `Download Image` (Google Drive Node)
* **Acción:** Habilitar la continuación ante errores.
* **Propiedad a agregar/cambiar en JSON:**
  ```json
  "onError": "continueErrorOutput"
  ```
  *(Esto causará que si falla la descarga, el nodo continúe y devuelva un objeto con la propiedad `error` en lugar de detener el workflow)*

### 2.2. Incorporación de Nodo de Control `🔀 ¿Descarga Exitosa?` (IF Node)
* **Ubicación:** Entre `Download Image` y `Send WhatsApp`.
* **ID Propuesto:** `check-download-success`
* **Tipo:** `n8n-nodes-base.if`
* **Lógica:** Evaluar si el resultado anterior NO contiene un error.
  * **Condición:**
    * **Valor izquierdo:** `={{ !$json.error }}`
    * **Operación:** `equals` (boolean)
    * **Valor derecho:** `true`

### 2.3. Incorporación de Nodo `❌ Marcar como Fallido` (PostgreSQL Node)
* **Ubicación:** Conectado a la salida `false` (falla) del nodo `¿Descarga Exitosa?`.
* **ID Propuesto:** `mark-failed-db`
* **Tipo:** `n8n-nodes-base.postgres`
* **Credenciales:** `postgres_carmay` (ID: `qq6LeroBSIWTFfx2`)
* **Consulta SQL:**
  ```sql
  UPDATE whatsapp_queue SET estado = 'FALLIDO', fecha_envio = NOW() WHERE id = {{ $('Loop Batch').item.json.id }};
  ```

### 2.4. Reconexiones del Flujo
1. La salida de `Download Image` se conecta a `🔀 ¿Descarga Exitosa?`.
2. La salida **true** de `🔀 ¿Descarga Exitosa?` se conecta a `Send WhatsApp` (que prosigue a `Mark Completed`).
3. La salida **false** de `🔀 ¿Descarga Exitosa?` se conecta a `❌ Marcar como Fallido`.
4. Las salidas de `Mark Completed` (nodo original de éxito) y `❌ Marcar como Fallido` (nuevo de error) se reconectan de vuelta a `Loop Batch` (index 0) para procesar el siguiente ítem de la tanda.

### 2.5. Reactivación del Temporizador (`Schedule Trigger`)
* **Acción:** Habilitar el nodo cron para ejecutarse cada 30 minutos.
* **Propiedad en JSON:**
  ```json
  "disabled": false
  ```

## 3. Plan de Verificación

1. **Aplicación:** Actualizar el JSON de la base de datos de n8n para el workflow `yqOdEH1gMiuvUHN6`.
2. **Reinicio/Validación:** Comprobar la estructura en n8n mediante `n8n-mcp` o consultando la base de datos.
3. **Prueba de Ejecución:** Simular o esperar la corrida y verificar en `whatsapp_queue` que:
   - Los ítems válidos se marquen como `COMPLETADO`.
   - Los ítems con Google Drive ID inválido se marquen como `FALLIDO` sin detener la ejecución de los siguientes.
