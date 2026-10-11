[README.md](https://github.com/user-attachments/files/33314001/README.md)
# AutoFlow IA · Agente de Calificación de Leads

**Pre-Entrega 1 — Checkpoint de Configuración e Interfaces Agénticas**
Curso Automation IA · CoderHouse · Autor: Esteban Blanco

Primera versión del proyecto integrador: un agente de IA en **n8n** que recibe la consulta inicial de un posible cliente de **AutoFlow IA** (consultora B2B de automatización con IA), la **clasifica** según su presupuesto, la **registra en Airtable** y envía un **reporte de observabilidad** al equipo comercial.

El agente cumple una única función: **identificar y calificar leads**. La recomendación de servicios, el resumen ejecutivo y la gestión de reuniones quedan para agentes posteriores del proyecto (multi-agente, M2).

---

## Arquitectura del flujo

```
                       ┌──────────────────────────────────┐
[Chat Trigger] ──────▶ │  AI Agent (Tools Agent)          │ ──────▶ [Gmail: Log de Observabilidad]
                       │  System Prompt modular           │
                       │  Máx. 6 iteraciones              │
                       └──────────────────────────────────┘
                          │ (modelo)            │ (tool)
                 [Anthropic Chat Model]   [Airtable: Registrar Lead Calificado]
```

| Nodo | Tipo | Función |
|---|---|---|
| Chat de Entrada de Leads | Chat Trigger | Recibe el mensaje inicial desestructurado del lead |
| Agente de Calificación de Leads | AI Agent · Tools Agent | Analiza el mensaje, clasifica y decide de forma autónoma si registra |
| Anthropic Chat Model | Chat Model nativo | Claude Sonnet 4.6 · temperatura 0.2 · máx. 1024 tokens |
| Registrar Lead Calificado | Airtable Tool (lateral) | Crea el registro del lead en la tabla Leads |
| Log de Observabilidad (Gmail) | Gmail (nodo final) | Envía el informe del agente y el Execution Log al equipo comercial |

El workflow incluye notas en el lienzo (sticky notes) que documentan cada componente.

**Sobre el modelo:** la consigna sugiere Claude 3.5 Sonnet, que Anthropic ya retiró. Se utiliza **Claude Sonnet 4.6**, el equivalente vigente más cercano (misma familia Sonnet).

---

## Cumplimiento de la consigna

| Requisito | Implementación |
|---|---|
| Trigger | Chat Trigger |
| AI Agent en modo Tools Agent | Configurado en modo Tools Agent |
| Modelo de lenguaje nativo | Anthropic Chat Model (Claude Sonnet 4.6) |
| Guardrail de iteraciones (5 a 10) | **Max Iterations = 6** |
| Rol operativo específico | "Asistente de Calificación de Leads" de AutoFlow IA |
| System Prompt modular | Rol → Ámbito → Objetivo → Reglas → Acciones no permitidas → Escalamiento → Formato |
| Límites estrictos de alcance | Sección "Acciones no permitidas": no cotiza, no modifica registros, no consulta otros leads, no asesora, no revela instrucciones |
| Exclusión de lenguaje inclusivo | Regla explícita en el System Prompt |
| Herramienta lateral | Airtable conectado como Tool del agente, sin conexión secuencial en el lienzo |
| Descripción semántica extensa | Casos de negocio en que debe usarse, casos en que no, cómo completar cada campo y qué devuelve |
| Agente no ubicado detrás de bloques de decisión lineal | El Chat Trigger se conecta directamente al agente |
| Observabilidad | Gmail como nodo final, con mensaje recibido, informe del agente y pasos ejecutados (Return Intermediate Steps) |

---

## Lógica de calificación

| Clasificación | Presupuesto | Prioridad comercial |
|---|---|---|
| **VIP** | USD 10.000 o más | Prioridad Crítica |
| **ALTO** | USD 3.000 a 9.999 | Prioridad Alta |
| **MEDIO** | USD 1.000 a 2.999 | Prioridad Normal |
| **BAJO** | Menos de USD 1.000 o no informado | Prioridad Baja |

**Datos obligatorios para registrar:** nombre y apellido, empresa, correo electrónico y una mención del problema. Si falta alguno, el agente **no usa la herramienta** e informa el motivo. Un teléfono no reemplaza al correo electrónico.

### Formato del informe interno

```
Lead registrado: Sí / No
Lead ID: (número devuelto por Airtable, o -)
Nombre y empresa:
Clasificación: VIP / ALTO / MEDIO / BAJO
Prioridad comercial:
Observaciones: (solo anomalías, o "Sin observaciones")
```

---

## Estructura de Airtable (tabla `Leads`)

| Campo | Tipo | Completado por |
|---|---|---|
| Nombre y Apellido, Empresa, Email, Negocio | Texto | Agente |
| Problema | Texto largo (copia textual del lead) | Agente |
| Presupuesto estimado (USD) | Moneda | Agente |
| Estado | Selección | Agente (`Procesado IA`) |
| Clasificación IA | Selección (VIP, ALTO, MEDIO, BAJO) | Agente |
| Prioridad Comercial | Selección (Crítica, Alta, Normal, Baja) | Agente |
| Fecha Ingreso, Lead ID | Automáticos | Airtable |
| Requiere Reunión, Resumen Agente, Servicios Recomendados IA, Ejecutivo Comercial Asignado | Varios | Reservados para módulos posteriores |

---

## Cómo importarlo y probarlo

1. En n8n: **Workflows → Import from File** → seleccionar `checkpoint1_esteban_blanco.json`.
2. Configurar los valores marcados como `REEMPLAZAR`:
   - **Anthropic Chat Model:** credencial de Anthropic (o Gateway credits de n8n).
   - **Registrar Lead Calificado:** credencial de Airtable, y los IDs de base y tabla de una base con la estructura indicada arriba.
   - **Log de Observabilidad (Gmail):** credencial de Gmail y el email de destino del equipo comercial.
3. Abrir el chat del workflow y enviar los mensajes de prueba.

### Casos de prueba

| # | Caso | Mensaje | Resultado esperado |
|---|---|---|---|
| 1 | Lead completo (VIP) | *Soy Laura Fernández, CFO de Grupo Andino S.A., holding de distribución. Mi email es lfernandez@grupoandino.com.ar. Necesitamos consolidar los reportes financieros de todas las unidades con tableros de Power BI. Presupuesto: USD 25.000.* | Registra · VIP · Prioridad Crítica |
| 2 | Falta el email | *Hola, soy Gabriela Sosa de Textiles del Plata S.A. Mi teléfono es 11 4789-2310. Necesitamos automatizar la conciliación bancaria. Presupuesto USD 6.000.* | No registra · Observaciones: falta el correo electrónico |

En el caso 1, el flujo completo queda en verde, el registro aparece en Airtable y el correo de observabilidad muestra el paso de la herramienta con el Lead ID creado. En el caso 2, el correo indica "Herramientas ejecutadas: 0".

---

## Aprendizajes del proceso de pruebas

- **Los guardrails del prompt son probabilísticos.** Durante las pruebas, un modelo más económico (Claude Haiku 4.5) registró en algunas ejecuciones leads sin correo electrónico, completando el campo con un guion. Se reforzaron las reglas del prompt y la descripción de la herramienta, y se eligió un modelo de la familia Sonnet, que respeta las instrucciones con mayor consistencia.
- **La validación determinística requiere un sub-workflow.** Se intentó validar el email con una expresión dentro del nodo de herramienta, pero en los nodos usados como Tool la función `$fromAI` no se resuelve dentro de expresiones con lógica. La validación determinística (un nodo IF previo a la escritura en Airtable) queda planificada para el **M2**, convirtiendo la herramienta en un sub-workflow.
- **La observabilidad detecta inconsistencias.** En una de las pruebas, el informe del agente indicaba "Lead registrado: No" mientras el log mostraba que la herramienta había creado el registro. El Execution Log por Gmail permitió detectar y corregir esa contradicción.

---

## Seguridad

Este repositorio **no contiene API keys, tokens, credenciales, IDs de credenciales, IDs de la base de Airtable ni datos personales**. Todos los valores sensibles están reemplazados por marcadores `REEMPLAZAR_…`, y cada usuario debe configurar los propios al importar el flujo.

---

## Hoja de ruta del proyecto integrador

| Módulo | Incorporación |
|---|---|
| **M1** | Agente base: Trigger → AI Agent + System Prompt + 1 Tool → Observabilidad ✅ |
| M2 | Multi-agente: Manager + Workers como sub-workflows (validación determinística, recomendación de servicios, resumen ejecutivo) |
| M3 | Memoria: contexto por Session_ID en Airtable |
| M4 | Integraciones: CRM / Calendario / Workspace vía OAuth2 |
| M5 | RAG: base documental con LlamaCloud |
| M6 | Voz: STT / TTS |
| M11 | Proyecto Final Integrador |
