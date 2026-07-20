# Requisitos del Proyecto (TODO List)

Este archivo registra las características y cambios planificados, aprobados, en proceso y completados en el proyecto.

## Características Completadas

- [x] **Configuración de memoria de intercambio (Swap):** Configurado y verificado en el VPS para prevenir errores de Out of Memory (OOM) causados por Puppeteer.
- [x] **Migración a SSH para la ejecución de agy CLI:** Cambio del nodo `🧠 Invocar agy CLI` a tipo `ssh` para ejecutar el script directamente sobre la máquina host del VPS de forma nativa.
- [x] **Acceso de la IA a Docker:** Asignación del usuario `carmay-dev` al grupo `docker` en el VPS, permitiendo ejecutar y monitorear contenedores a través de `sg docker`.

- [x] **Desbloqueo de escritura en disco en n8n:** Descomentada la variable `N8N_RESTRICT_FILE_ACCESS_TO=/data` en `/opt/n8ncarmay/docker-compose-vps.yml` y reiniciado el stack de contenedores para aplicar los cambios de permisos.

## Características Pendientes
- [ ] Implementar el paquete `auto skills` de `midu dev`.
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
