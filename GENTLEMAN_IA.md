# Ecosistema de Trabajo IA - Gentleman Programming & Engram

Este proyecto utiliza la metodología **SDD (Spec-Driven Development)** y memoria persistente **Engram** de manera obligatoria para guiar la interacción con los agentes de IA.

## Reglas Obligatorias para el Agente de IA

### 1. Memoria Persistente (Engram)
- Antes de iniciar cualquier tarea, debes consultar la memoria reciente del proyecto usando `engram context` o realizar búsquedas con `engram search`.
- Al finalizar una decisión técnica importante, resolver un bug complejo o cambiar configuraciones, debes guardar el aprendizaje inmediatamente ejecutando:
  `engram save "Título descriptivo" "Detalles técnicos del cambio, por qué y dónde se realizó" --project Gestor`
- Al final de la sesión, escribe un resumen con el formato de Engram (`## Goal`, `## Discoveries`, `## Accomplished`, `## Next Steps`).

### 2. Spec-Driven Development (SDD)
- **Prohibido escribir código directamente** sin antes crear o consultar una especificación.
- Si vas a implementar una nueva característica o refactorización:
  1. Analiza el codebase (Scout).
  2. Crea un archivo de especificación (ej. `docs/specs/feature-name.spec.md` o similar).
  3. Pide la aprobación del usuario antes de proceder a la escritura de código.

### 3. Habilidades y Estructura
- Respeta los patrones arquitectónicos y las guías definidas en el archivo de especificaciones y en los documentos de diseño del repositorio.
