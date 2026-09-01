# Requisitos del Proyecto (TODO List)

Este archivo registra las características y cambios planificados, aprobados, en proceso y completados en el proyecto.

## Características Completadas

- [x] **Configuración de memoria de intercambio (Swap):** Configurado y verificado en el VPS para prevenir errores de Out of Memory (OOM) causados por Puppeteer.
- [x] **Migración a SSH para la ejecución de agy CLI:** Cambio del nodo `🧠 Invocar agy CLI` a tipo `ssh` para ejecutar el script directamente sobre la máquina host del VPS de forma nativa.
- [x] **Acceso de la IA a Docker:** Asignación del usuario `carmay-dev` al grupo `docker` en el VPS, permitiendo ejecutar y monitorear contenedores a través de `sg docker`.
- [x] **Desbloqueo de escritura en disco en n8n:** Descomentada la variable `N8N_RESTRICT_FILE_ACCESS_TO=/data` en `/opt/n8ncarmay/docker-compose-vps.yml` y reiniciado el stack de contenedores para aplicar los cambios de permisos.
- [x] **Renombrar proyecto a Gestor:** Renombrado el proyecto de `facebookBOT` a `Gestor` en la configuración de Engram, reglas locales y documentación técnica.
- [x] **Robustez y Activación de Gestor WhatsApp:**
  - [x] Habilitar `"onError": "continueErrorOutput"` en el nodo `Download Image`.
  - [x] Agregar nodo condicional `🔀 ¿Descarga Exitosa?` para validar errores de descarga.
  - [x] Agregar nodo `❌ Marcar como Fallido` en Postgres para la rama de fallos.
  - [x] Reconectar el flujo para reanudar el bucle tras un error.
  - [x] Reactivar el temporizador automático `Schedule Trigger` (`"disabled": false`).
- [x] **Prevenir solapamiento de ejecuciones (Anti-Duplicados):** Configuración `"concurrency": 1` en los ajustes del workflow para que n8n bloquee nuevas ejecuciones mientras haya una activa.

## Características Pendientes

### En Proceso (2026-08-17)
- [x] **Auditoría y corrección de flujos Facebook y WhatsApp (Marketing Expert):**
  - [x] Corregir nodo `🧠 Invocar Proxy Gemini`: URL de `localhost:20128/omni/api/v1/chat/completions` a `172.16.1.1:20128/v1/chat/completions`
  - [x] Configurar modelo por defecto a `gemini/gemini-3.5-flash` en OmniRoute
  - [x] Validar resultado de ejecución de Playwright en `Gestor Facebook` (`fbPosterWorkflow`) antes de actualizar la BD (`COMPLETADO` vs `FALLIDO`)
  - [x] Optimizar la eliminación automática de archivos temporales y capturas de pantalla de Playwright (`pw_step*.png`) tras cada publicación
  - [x] Confirmar regla global de envío de solo imagen sin texto directo para todas las redes (contenido visual autosuficiente)
  - [x] **Límites de Seguridad Anti-Spam en WhatsApp (`yqOdEH1gMiuvUHN6`):**
    - [x] Límite máximo estricto de **30 fotos diarias** (`sent_at >= CURRENT_DATE AT TIME ZONE 'America/Havana' < 30`).
    - [x] Horario de restricción nocturna: **Prohibido enviar entre 12:00 AM y 7:00 AM (Hora Cuba)** (`EXTRACT(HOUR) >= 7`).
    - [x] Reducción de ráfagas a **2-3 fotos por lote** (`LIMIT (2 + floor(random() * 2))`).
    - [x] Reducción de frecuencia a **1 vez cada 60 minutos** y demoras orgánicas de **60 a 180 segundos** entre fotos.

### Completadas
- [x] **Validación del flujo orquestador central:**
  - [x] Comprobar que el nodo `💾 Guardar contexto.json` guarde el archivo `/data/tmp_post/contexto.json` correctamente y sin errores de permisos.
  - [x] Validar que `🧠 Invocar agy CLI` pueda procesar la decisión y continuar con el enrutamiento multicanal.
- [x] **Validar ejecución del nodo 🧠 Invocar agy CLI**
  - [x] Ejecutar workflow de prueba y revisar logs para asegurar que no exceda 8 minutos.
  - [x] Corregir configuración de tiempo límite y permisos si es necesario.
- [x] **Configurar tiempo máximo de ejecución a 3 min en el nodo SSH**
  - [x] Ajustar el parámetro de timeout del nodo SSH `🧠 Invocar agy CLI` en n8n a 180 segundos.
  - [x] Ajustar llamada del comando agregando redireccionamiento de salida seguro, ejecución como usuario `carmay-dev` y forzado por señal KILL.
  - [x] Verificar que el nodo SSH aborte automáticamente tras 3 min si no hay respuesta.
- [x] **Analizar causa del bloqueo de 8 min del nodo `Invocar agy CLI`**
  - [x] Revisar logs de ejecución y buscar errores de tiempo de espera o permisos.
  - [x] Ajustar configuración de timeout del nodo o del workflow.
- [x] **Asegurar que `contextp.json` se escribe con permisos correctos**
  - [x] Verificar directorio `/data/tmp_post` y permisos de escritura.
  - [x] Modificar ruta o permisos si es necesario.
