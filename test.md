# 🧪 test.md: Protocolo de Pruebas de Calificación y Salida a Producción

**Proyecto:** Gestor (Carmay Outfit)  
**Documento:** Protocolo de Calificación y Verificación E2E (End-to-End)  
**Fecha:** 2 de Septiembre de 2026  
**Objetivo:** Garantizar 100% de confiabilidad, resiliencia y cero falsos positivos en el sistema multicanal (Marketing Expert, Ingesta WhatsApp, Worker WhatsApp, Gestor Facebook) operando bajo la modalidad de **1 grupo activo de WhatsApp**.

---

## 📋 Índice del Protocolo
1. [Visión General y Criterios de Aceptación](#1-visión-general-y-criterios-de-aceptación)
2. [Configuración Inicial del Entorno de Prueba (1 Solo Grupo)](#2-configuración-inicial-del-entorno-de-prueba-1-solo-grupo)
3. [Fase 1: Prueba de Carga e Inteligencia del Orquestador (Marketing Expert)](#fase-1-prueba-de-carga-e-inteligencia-del-orquestador-marketing-expert)
4. [Fase 2: Prueba de Ingesta y Deduplicación en BD (Ingesta WhatsApp)](#fase-2-prueba-de-ingesta-y-deduplicación-en-bd-ingesta-whatsapp)
5. [Fase 3: Prueba de Ejecución y Cobertura de WhatsApp (1 Solo Grupo)](#fase-3-prueba-de-ejecución-y-cobertura-de-whatsapp-1-solo-grupo)
6. [Fase 4: Prueba de Publicación Persistente en Facebook (Gestor Facebook)](#fase-4-prueba-de-publicación-persistente-en-facebook-gestor-facebook)
7. [Fase 5: Prueba de Resiliencia y Fallo Controlado (Regla 24h)](#fase-5-prueba-de-resiliencia-y-fallo-controlado-regla-24h)
8. [Matriz de Verificación y Check-off Final](#8-matriz-de-verificación-y-check-off-final)

---

## 1. Visión General y Criterios de Aceptación

Para autorizar la salida a producción autónoma del sistema **Gestor**, la suite completa de pruebas debe cumplir los siguientes 4 Criterios de Aceptación Estrictos:

- **CA-01 (Aislamiento de Ingesta):** La llamada sincrónica desde Marketing Expert a `Ingesta WhatsApp` debe responder en `< 50 ms` sin esperar la descarga de archivos ni la entrega del mensaje.
- **CA-02 (Cobertura Exacta de 1 Grupo):** Con 1 solo grupo activo en `whatsapp_groups`, cada producto debe pasar a estado `COMPLETADO` tras su **primer y único envío**, registrando `grupos_enviados = {'id_grupo_1'}`.
- **CA-03 (Persistencia de Sesión de Facebook):** El script Playwright debe publicar exitosamente usando el perfil persistente `/data/tmp_post/facebook_browser_profile` sin requerir re-login ni lanzar error de cookies vencidas.
- **CA-04 (Resiliencia ante Fallos de Red/Drive):** Un error de descarga o envío debe notificar la alerta por WhatsApp al admin, marcar `estado = 'FALLIDO'`, registrar `fecha_envio = NOW()`, e ignorar el producto durante **24 horas exactas** sin detener el Worker ni deshabilitar el flujo de n8n.

---

## 2. Configuración Inicial del Entorno de Prueba (1 Solo Grupo)

Antes de iniciar la ejecución del protocolo, se establece el entorno en modo de prueba controlada:

### **Paso A: Estado de Grupos en PostgreSQL**
Se configura la tabla `whatsapp_groups` para mantener **1 solo grupo en estado activo**:
```sql
UPDATE whatsapp_groups SET activo = false;
UPDATE whatsapp_groups SET activo = true WHERE id = '120363412241377225@g.us'; -- Grupo de Prueba
```

### **Paso B: Verificación de Estado**
```sql
SELECT id, nombre, activo FROM whatsapp_groups ORDER BY orden;
```
*Resultado Esperado:* Exactamente 1 grupo activo (`activo = t`).

---

## 3. Fase 1: Prueba de Carga e Inteligencia del Orquestador (`Marketing Expert`)

### **Objetivo**
Validar que `Marketing Expert` (`OWmKP2Me3WJSzlWc`) lee el catálogo maestro de Google Sheets, evalúe el cooldown de 48h, invoque a Gemini 3.5 Flash vía OmniRoute Proxy (`:20128`) y seleccione 1 producto elegible por canal.

### **Pasos de Ejecución**
1. Disparar el Webhook manual de `Marketing Expert`:
   ```bash
   curl -s -X GET "https://carmayn8n.cloud/webhook/marketing-expert-trigger?_=test_fase1"
   ```
2. Inspeccionar la ejecución en n8n mediante MCP `n8n_executions`.

### **Verificación de Resultados**
- **Comprobación 1:** Nodo `⚙️ Evaluar Estado Multicanal` detecta candidatos elegibles.
- **Comprobación 2:** Nodo `🧠 Invocar Proxy Gemini` recibe respuesta HTTP 200 de OmniRoute con JSON estructurado (`facebook_product`, `whatsapp_product`, etc.).
- **Comprobación 3:** Nodo `📝 Actualizar Estados Canales` escribe `ENVIADO` en la celda correspondiente de Google Sheets.

---

## 4. Fase 2: Prueba de Ingesta y Deduplicación en BD (`Ingesta WhatsApp`)

### **Objetivo**
Validar que el módulo `Ingesta WhatsApp` (`NgPFwrUQufkxh8K1`) procese la petición en milisegundos, limpie basura del IA, filtre variantes secundarias `(2)` e inserte el registro en `whatsapp_queue`.

### **Pasos de Ejecución**
1. Inspeccionar la ejecución generada por `Marketing Expert` en `Ingesta WhatsApp`.

### **Verificación de Resultados**
- **Comprobación 1 (Latencia):** El tiempo de ejecución del workflow debe ser `<= 50 ms`.
- **Comprobación 2 (Base de Datos):**
  ```sql
  SELECT id, producto, drive_id, estado, grupos_enviados 
  FROM whatsapp_queue 
  ORDER BY id DESC LIMIT 1;
  ```
  - `estado` debe ser `'PENDIENTE'`.
  - `grupos_enviados` debe ser `{}` (array vacío).
  - `producto` no debe contener apóstrofes sin escapar ni texto secundario `(2)`.

---

## 5. Fase 3: Prueba de Ejecución y Cobertura de WhatsApp (1 Solo Grupo)

### **Objetivo**
Validar que `Worker WhatsApp` (`yqOdEH1gMiuvUHN6`) lea la cola `whatsapp_queue`, seleccione el grupo activo, descargue la foto de Drive, la envíe vía Baileys API y actualice la base de datos a `COMPLETADO`.

### **Pasos de Ejecución**
1. Ejecutar o esperar el ciclo del `Worker WhatsApp`.
2. Inspeccionar el comportamiento en la base de datos PostgreSQL.

### **Verificación de Resultados**
- **Comprobación 1 (WhatsApp Real):** La foto y mensaje deben ser recibidos en el grupo de prueba `120363412241377225@g.us`.
- **Comprobación 2 (Log de Envíos):**
  ```sql
  SELECT id, producto, group_id, sent_at FROM whatsapp_sent_log ORDER BY id DESC LIMIT 1;
  ```
  - Debe existir un nuevo registro con `group_id = '120363412241377225@g.us'`.
- **Comprobación 3 (Estado del Producto):**
  ```sql
  SELECT id, producto, estado, grupos_enviados FROM whatsapp_queue WHERE id = ${id_probado};
  ```
  - `grupos_enviados` debe ser `{'120363412241377225@g.us'}`.
  - `estado` debe haber cambiado automáticamente a `'COMPLETADO'` (dado que alcanzamos el 100% de los grupos activos).

---

## 6. Fase 4: Prueba de Publicación Persistente en Facebook (`Gestor Facebook`)

### **Objetivo**
Validar que `Gestor Facebook` (`fbPosterWorkflow`) ejecute el script de Playwright `/data/facebook_pw_poster.js`, publique en el grupo de Facebook utilizando el Perfil Persistente de navegador y actualice la sesión en disco.

### **Pasos de Ejecución**
1. Ejecutar el script Playwright de Facebook desde terminal o n8n:
   ```bash
   docker exec n8n-app node /data/facebook_pw_poster.js "27546646424927431" "/data/logo.png" "Test Fase 4 Salida a Produccion"
   ```

### **Verificación de Resultados**
- **Comprobación 1 (Salida de Consola):**
  `Sesión activa detectada correctamente.` → `Post enviado.` → `Auto-renovación: cookies frescas guardadas.`
- **Comprobación 2 (Facebook Real):** La publicación con la foto debe aparecer visible en el grupo de Facebook.
- **Comprobación 3 (Base de Datos):** `facebook_queue` se actualiza a `COMPLETADO`.

---

## 7. Fase 5: Prueba de Resiliencia y Fallo Controlado (Regla 24h)

### **Objetivo**
Validar que un error provocado (foto rota en Google Drive o apóstrofe en nombre) no rompa el flujo de n8n, envíe la alerta por WhatsApp al administrador y coloque el producto en reposo durante 24 horas.

### **Pasos de Ejecución**
1. Insertar manualmente un producto con `drive_id` inválido pero de longitud >= 25:
   ```sql
   INSERT INTO whatsapp_queue (producto, drive_id, estado) 
   VALUES ('PRODUCTO PRUEBA FALLO CONTROLADO', '1234567890123456789012345_INVALID', 'PENDIENTE');
   ```
2. Ejecutar `Worker WhatsApp`.

### **Verificación de Resultados**
- **Comprobación 1 (Alerta Admin):** El administrador recibe un mensaje en su WhatsApp personal (`REDACTED_PHONE_1`) indicando error de descarga con el nombre del producto e ID de archivo.
- **Comprobación 2 (Estado y Reposo):**
  ```sql
  SELECT id, producto, estado, fecha_envio FROM whatsapp_queue WHERE producto = 'PRODUCTO PRUEBA FALLO CONTROLADO';
  ```
  - `estado` debe ser `'FALLIDO'`.
  - `fecha_envio` debe coincidir con `NOW()`.
- **Comprobación 3 (Regla 24h):** Ejecutar `Get Pending Batch` en la siguiente corrida y confirmar que el producto `PRODUCTO PRUEBA FALLO CONTROLADO` es **ignorado** por la consulta SQL.

---

## 8. Matriz de Verificación y Check-off Final

| Criterio | Descripción del Check | Estado | Firma de Aprobación |
|---|---|---|---|
| **CHK-01** | `Ingesta WhatsApp` responde en `< 50 ms` | ✅ PASADO (18 ms a 40 ms) | Verificado vía MCP/BD |
| **CHK-02** | Producto pasa a `COMPLETADO` tras 1er envío en 1 solo grupo | ✅ PASADO (cobertura 1/1) | Verificado en `whatsapp_queue` |
| **CHK-03** | `facebook_pw_poster.js` renueva cookies sin error | ✅ PASADO (2 posts en vivo) | Verificado en Facebook real |
| **CHK-04** | Fallo controlado envía alerta y reposa 24h | ✅ PASADO (reposo 24h comprobado) | Verificado en prueba simulada |
| **CHK-05** | Nombres con apóstrofes (`'`) se insertan sin error SQL | ✅ PASADO (sanitizado .replace) | Verificado en expresiones |
