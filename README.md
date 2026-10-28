# Gestor — Hub de Redes Sociales y Marketing Automatizado

Sistema de distribución multicanal de productos para Facebook, Instagram, Telegram y WhatsApp con generación asistida por modelos LLM.

## Contenido

- `workflow_marketing_expert.json`: Orquestador principal de generación de copys y enrutamiento.
- `workflow_ingesta_whatsapp.json`: Recepción y encolado de mensajes para WhatsApp.
- `workflow_worker_whatsapp.json`: Procesador de colas de envío hacia Baileys.
- `DOCUMENTACION_FLUJOS_GESTOR.md`: Manual operativo y técnico de los flujos.

## Despliegue Portable (Docker)

1. Crear el archivo `.env` a partir de `.env.example`:
   ```bash
   cp .env.example .env
   ```
2. Rellenar las variables con las claves y tokens correspondientes.
3. Levantar el servicio:
   ```bash
   docker compose up -d
   ```
