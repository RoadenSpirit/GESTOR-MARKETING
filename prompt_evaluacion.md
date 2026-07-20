# 📊 Evaluación de Prompts y Flujo de Trabajo Agéntico

## Proyecto: facebookBOT
## Fecha de Evaluación: 6 de julio, 2026
## Evaluador: Roberto (AI Workflow Analyst)

---

## Resumen Ejecutivo

Este informe presenta una evaluación detallada de los prompts y las prácticas de interacción agéntica del desarrollador durante el trabajo en el proyecto **facebookBOT**. El análisis se basa en el historial de conversaciones extraído del archivo de logs `transcript.jsonl` del Conversation ID `6d0a60e2-9a4c-4cd3-a6b6-95666892e452`.

El usuario demuestra una **madurez técnica alta** y un entendimiento sólido de las limitaciones de los modelos de lenguaje (LLM). Destaca su capacidad para evitar la complacencia (el sesgo de aceptar todo lo que la IA propone), su rigidez para evitar antipatrones de desarrollo (como el uso excesivo de nodos de código para evadir problemas de configuración), y su excelente gestión de control de daños y reversión cuando comete un error de contexto.

*   **Puntuación General:** **8.1 / 10**
*   **Fortaleza Clave:** Escepticismo saludable y control de alcance ("Blast Radius").
*   **Debilidad Clave:** Presencia frecuente de errores tipográficos en los prompts que podrían inducir a error en contextos más ambiguos, y falta de una especificación técnica (`SPEC.md`) previa al inicio del desarrollo.

---

## Metodología

La evaluación se realizó mediante el análisis sistemático de los mensajes del usuario (`USER_INPUT`) y las respuestas del agente en el historial de la conversación. Las métricas y mejores prácticas aplicadas corresponden a los estándares de **Agentic Engineering (Ingeniería Agéntica)** y **Desarrollo Guiado por Especificaciones (Spec-Driven Development - SDD)** vigentes en **2025-2026**, enfocados en la predictibilidad, control de contexto y estructuración del flujo de trabajo con asistentes de código.

---

## Análisis del Historial de Conversaciones

Durante la conversación, el desarrollador y el agente trabajaron en resolver problemas de infraestructura y configuración de un flujo de automatización en **n8n** y herramientas CLI asociadas (`agy`):
1.  **Optimización del VPS:** Configuración de memoria de intercambio (swap) para evitar errores OOM causados por Puppeteer.
2.  **Integración y ejecución nativa:** Resolución de problemas de timeout de `agy` al ser invocado por n8n, evaluando opciones de SSH y local.
3.  **Permisos y Docker:** Corrección de problemas de escritura de archivos (`contexto.json`) mediante la habilitación de accesos del usuario `carmay-dev` al socket de Docker y ajuste de restricciones de n8n en `docker-compose-vps.yml`.
4.  **Recuperación ante errores de contexto:** Reversión completa y aislamiento de un prompt largo sobre un proyecto de diseño web enviado accidentalmente.

---

## Evaluación por Dimensiones

### a) Claridad y Especificidad de los Prompts
*   **Puntuación: 7 / 10**
*   **Comentario:** Los prompts son específicos en cuanto a los errores técnicos que se intentan resolver (proporciona logs de error completos con trazas de pila). Sin embargo, contienen abundantes errores ortográficos y de digitación ("compelta", "revicio nsi", "permisis", "neceistamso", "ultiam", etc.). Aunque la IA los interpreta correctamente gracias a su tolerancia al ruido, esto introduce un riesgo de ambigüedad en proyectos complejos.
*   **Ejemplo:**
    > *"analiza la ultiam ejecuccion ocurrion algu nerror el nodo invocar agy estaba ejecutandose por mas de 8 min algun error debio ocurrir revisa"* (Prompt 20)

### b) Gestión del Contexto
*   **Puntuación: 8 / 10**
*   **Comentario:** El usuario aporta un contexto excelente mediante la compartición de mensajes de error íntegros de n8n. Mantiene el hilo del problema activo y utiliza un archivo `requisitos.md` como memoria externa persistente para evitar la pérdida de contexto ("context drift").
*   **Ejemplo:** Proporcionar el log estructurado completo de error en los Prompts 04 y 10 para guiar el diagnóstico del agente de forma precisa.

### c) Control de Alcance (Scope Control)
*   **Puntuación: 8 / 10**
*   **Comentario:** Excelente. El desarrollador limita activamente el radio de acción del agente para evitar modificaciones masivas descontroladas ("vibe coding"). Exige comandos explícitos y explicaciones de impacto antes de permitir cualquier ejecución.
*   **Ejemplo:**
    > *"nada de crear neuvos flujos diem que necesitas cofngiurar lienas exactas o preferiblement ecomandos para ejcutar y que impaco tednran y proque neceistamso ahcerlo0s"* (Prompt 13)

### d) Manejo de Errores y Recuperación
*   **Puntuación: 9 / 10**
*   **Comentario:** Sobresaliente. Cuando el usuario se equivocó e ingresó un prompt de revisión web para un proyecto ajeno, no intentó "parchear" la conversación. Pidió encapsular el error en un archivo `prompt_error.md`, revertir los cambios al estado anterior usando la documentación de `requisitos.md`, y restablecer la línea base del proyecto original.
*   **Ejemplo:**
    > *"he coemtido un gran error el promt que te di fue una revicion de un proyecto diferente al que estabamos amnejando encpsual mi promt en un archiv .md guardadlo en la carpeta del proyecto web y desace todos los cambios..."* (Prompt 26)

### e) Enfoque Guiado por Especificaciones (Spec-Driven / Plan-First)
*   **Puntuación: 8 / 10**
*   **Comentario:** Utiliza archivos de planes/especificaciones como `sdd_ssh_node_change.md` y `requisitos.md` para coordinar el trabajo. El agente debe proponer un plan en markdown, el usuario lo aprueba en el chat (`[Approved] ...`) y luego se ejecuta. Esto sigue los estándares modernos de ingeniería agéntica.
*   **Ejemplo:** El flujo de aprobación explícita del plan de migración SSH en el Prompt 09.

### f) Madurez del Flujo de Trabajo Agéntico
*   **Puntuación: 8.5 / 10**
*   **Comentario:** El desarrollador demuestra saber que el agente no es infalible y que tiende a tomar el "camino fácil" (como proponer un nodo Code temporal en n8n para evadir problemas de permisos reales). El usuario rechaza este comportamiento, catalogándolo correctamente como una "mala práctica" y forzando a ir a la raíz del problema (permisos en Docker/SO).
*   **Ejemplo:**
    > *"no me termian de convencer el sistema que propones una nod ocode para sustituir un nodo nativo de n8n es una muy mala practica debemos resolver donde guarda n8n los archivpos y configurar correctamente sus permisis..."* (Prompt 12)

### g) Comunicación Técnica
*   **Puntuación: 8 / 10**
*   **Comentario:** El usuario maneja terminología de infraestructura (SSH, Docker, permisos de escritura, sockets, timeouts, buffers) con fluidez. Sin embargo, al redactar comandos manuales, a veces introduce errores que consulta con la IA antes de ejecutar.
*   **Ejemplo:**
    > *"root@srv1571201:~# sudo sed -i 's/# - N8N_RESTICT_FILE_ACCESS_TO/- N8N_RESTRICT_FILE_ACCESS_TO/g'/    /opt/n8ncarmay/docker-compose.yml , esta bien escrito ????"* (Prompt 16) - *Nota: El comando original tenía un error de sintaxis en la ruta y una errata en la variable, corregido gracias a la verificación previa.*

---

## Fortalezas Identificadas

1.  **Cero Tolerancia a Parches Sucios ("Workarounds"):** Rechazo firme a soluciones rápidas pero técnicamente insostenibles (p. ej., evitar el uso de nodos Code innecesarios cuando el problema es de permisos del sistema de archivos).
2.  **Escepticismo Activo:** Exigencia de verificación y consulta documental externa en lugar de aceptar la primera sugerencia del agente.
3.  **Encapsulación de Errores:** Reversión y aislamiento impecables tras la entrada de un prompt erróneo, evitando "contaminar" el contexto actual de la ventana de chat.
4.  **Uso de Aprobaciones Explícitas:** Consumo de archivos de diseño intermedios (`sdd_*.md`) para autorizar modificaciones del sistema.

---

## Áreas de Mejora Prioritarias

1.  **Reducir Errores de Escritura (Typos):** Aunque la IA es tolerante, los errores tipográficos constantes en nombres de variables o comandos (`N8N_RESTICT_...`) aumentan la probabilidad de fallos silenciosos en scripts autogenerados.
2.  **Redacción de Requisitos Preventiva:** El archivo `requisitos.md` se fue construyendo y actualizando de forma reactiva a medida que surgían problemas. Lo ideal bajo SDD es iniciar con el diseño conceptual antes de tocar el código.
3.  **Guiar el Diagnóstico con Hipótesis:** Cuando ocurrió el bloqueo de 8 minutos, el usuario indicó al agente que "se enfocaba en lo que no era", pero pudo haber sido más directivo sobre qué variables del entorno o llamadas de red sospechaba que causaban la espera.

---

## Recursos y Referencias

*   **Desarrollo Guiado por Especificaciones (Spec-Driven Development):** Principio clave para desacoplar el "qué/por qué" del "cómo".
*   **Prácticas de Gestión de Blast Radius:** Limitación de cambios en sistemas host y VPS mediante entornos controlados.
*   **Context Engineering:** Reducción de ruidos y mantenimiento de documentos de requisitos (`requisitos.md` / `SPEC.md`) para optimizar la ventana de contexto de los modelos frontera.

---

## Plan de Acción Sugerido

1.  **Escribir un `SPEC.md` Inicial:** Para el próximo módulo o flujo, redactar los requisitos técnicos, dependencias e infraestructura *antes* de iniciar el chat con el agente.
2.  **Establecer Checkpoints de Sintaxis:** Antes de ejecutar comandos destructivos o de edición rápida (`sed`, `chmod`, `rm`), usar herramientas linter o pedir confirmación formal en formato JSON/Markdown del cambio exacto.
3.  **Mantener la Arquitectura Limpia en n8n:** Continuar con el principio de resolver los problemas de permisos a nivel de Docker/SO en lugar de crear nodos de parche dentro del lienzo de n8n.
