[README.md](https://github.com/user-attachments/files/33314192/README.md)
# Pre-Entrega 1 · Agente Base de Calificación de Leads

**Checkpoint de Configuración e Interfaces Agénticas**
Curso Automation IA · CoderHouse · Autor: Esteban Blanco

Agente de IA en **n8n** que recibe la consulta de un posible cliente de **AutoFlow IA** (consultora B2B de automatización con IA), la **clasifica** según su presupuesto, la **registra en Airtable** y envía un **reporte de observabilidad** por Gmail al equipo comercial.

**Archivo entregable:** `checkpoint1_esteban_blanco.json`

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
| Log de Observabilidad (Gmail) | Gmail (nodo final) | Envía el correo "Tarea completada" con el informe del agente y el Execution Log |

El workflow incluye notas en el lienzo (sticky notes) que documentan cada componente.

**Sobre el modelo:** la consigna sugiere Claude 3.5 Sonnet, que Anthropic ya retiró. Se utiliza **Claude Sonnet 4.6**, el equivalente vigente más cercano (misma familia Sonnet).

---

## Criterios de evaluación

| Criterio | Peso | Cómo se cumple |
|---|---|---|
| Observabilidad y log externo de finalización | 35 % | Nodo final **Gmail** que envía al equipo comercial un correo **"Tarea completada"** con el informe del agente y el resultado de la herramienta |
| Arquitectura de Interfaces No-Code | 30 % | **Airtable** acoplado lateralmente como Tool del agente, sin conexiones secuenciales en el lienzo |
| Guardrails financieros y observabilidad | 35 % | **Max Iterations = 6** (rango 5 a 10) y log automático de auditoría (Execution Log) hacia Gmail |

## Cumplimiento de la consigna

| Requisito | Implementación |
|---|---|
| Trigger | Chat Trigger |
| AI Agent en modo Tools Agent | Configurado en modo Tools Agent |
| Modelo de lenguaje nativo | Anthropic Chat Model (Claude Sonnet 4.6) |
| Guardrail de iteraciones (5 a 10) | **Max Iterations = 6** |
| Rol operativo específico | "Asistente de Calificación de Leads" de AutoFlow IA |
| System Prompt modular | Rol → Ámbito → Objetivo → Reglas → Acciones no permitidas → Escalamiento → Formato |
| Límites estrictos de alcance | No cotiza, no modifica registros, no consulta otros leads, no asesora y no revela sus instrucciones |
| Exclusión de lenguaje inclusivo | Regla explícita en el System Prompt |
| Herramienta lateral | Airtable conectado como Tool del agente |
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

### Informe que genera el agente

```
Lead registrado: Sí / No
Lead ID: (número devuelto por Airtable, o -)
Nombre y empresa:
Clasificación: VIP / ALTO / MEDIO / BAJO
Prioridad comercial:
Observaciones: (solo anomalías, o "Sin observaciones")
```

### Campos que registra en Airtable (tabla `Leads`)

Nombre y Apellido · Empresa · Email · Negocio · Problema (copia textual) · Presupuesto estimado (USD) · Estado (`Procesado IA`) · Clasificación IA · Prioridad Comercial

---

## Cómo importarlo y probarlo

1. En n8n: **Workflows → Import from File** → seleccionar `checkpoint1_esteban_blanco.json`.
2. Configurar los valores marcados como `REEMPLAZAR`:
   - **Anthropic Chat Model:** credencial de Anthropic (o Gateway credits de n8n).
   - **Registrar Lead Calificado:** credencial de Airtable y los IDs de base y tabla de una base con los campos indicados arriba.
   - **Log de Observabilidad (Gmail):** credencial de Gmail y el email de destino.
3. Abrir el chat del workflow y enviar los mensajes de prueba.

### Casos de prueba

| # | Caso | Mensaje | Resultado esperado |
|---|---|---|---|
| 1 | Lead completo | *Soy Laura Fernández, CFO de Grupo Andino S.A., holding de distribución. Mi email es lfernandez@grupoandino.com.ar. Necesitamos consolidar los reportes financieros de todas las unidades con tableros de Power BI. Presupuesto: USD 25.000.* | Registra · VIP · Prioridad Crítica |
| 2 | Falta el email | *Hola, soy Gabriela Sosa de Textiles del Plata S.A. Mi teléfono es 11 4789-2310. Necesitamos automatizar la conciliación bancaria. Presupuesto USD 6.000.* | No registra · Observaciones: falta el correo electrónico |

En el caso 1, el flujo completo queda en verde, el registro aparece en Airtable y el correo "Tarea completada" muestra el paso de la herramienta con el Lead ID creado. En el caso 2, el correo indica "Herramientas ejecutadas: 0".

---

## Observaciones de las pruebas

- **Elección del modelo:** con un modelo más económico (Claude Haiku 4.5), el agente registró en algunas ejecuciones leads sin correo electrónico. Se reforzaron las reglas del prompt y la descripción de la herramienta, y se eligió un modelo Sonnet, que respeta las instrucciones con mayor consistencia.
- **Valor de la observabilidad:** en una prueba, el informe del agente indicaba "Lead registrado: No" mientras la herramienta había creado el registro. El Execution Log enviado por Gmail permitió detectar y corregir esa inconsistencia.

---

## Seguridad

Este repositorio **no contiene API keys, tokens, credenciales, IDs de credenciales, IDs de la base de Airtable ni datos personales**. Los valores sensibles están reemplazados por marcadores `REEMPLAZAR_…`, que cada usuario configura al importar el flujo.
