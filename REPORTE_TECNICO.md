# Smart Expense & Task Concierge — Reporte técnico, de arquitectura y de oportunidades de negocio

> **Divulgación de uso de IA:** Este material fue elaborado con asistencia de Claude (Anthropic) y revisado por Jesus Armando Tapia Gallegos.

| | |
|---|---|
| **Proyecto** | Smart Expense & Task Concierge |
| **Contexto** | Reto Agent Builders (Platzi) |
| **Repositorio** | https://github.com/atapia9/smart-expense-task-concierge-n8n |
| **Demo (1:46)** | https://youtu.be/LGV56TEeTR8 |
| **Plataforma** | n8n (self-hosted) |
| **Fecha del reporte** | 2026-10-16 |

---

## 1. Resumen ejecutivo

Smart Expense & Task Concierge es un agente de IA construido en n8n que convierte **un solo mensaje en lenguaje natural** en dos acciones concretas: **registrar un gasto o ahorro en Google Sheets** y **agendar un recordatorio en Google Calendar**.

Ejemplo de entrada:

> "Pagué 450 pesos de gasolina hoy y recuérdame revisar las facturas el viernes a las 10 am"

El agente detecta la intención, extrae monto, concepto, categoría y fecha, ejecuta una o ambas herramientas y responde con un resumen en viñetas. Soporta correcciones conversacionales gracias a su memoria ("la comida fueron 320, no 280").

**Propuesta de valor:** eliminar la fricción de saltar entre hojas de cálculo, calendario y notas, con un costo de operación de $0 en el prototipo (Gemini free tier + n8n self-hosted + Google Workspace personal).

---

## 2. Problema y usuario objetivo

- **Problema:** las personas anotan gastos "después" y olvidan pendientes porque cambiar entre Sheets, Calendar y notas rompe el flujo. El resultado son finanzas desordenadas y recordatorios perdidos.
- **Usuario objetivo:** freelancers, emprendedores y profesionales con agenda ocupada que ya usan Google Workspace.

---

## 3. Arquitectura

### 3.1 Diagrama lógico

```
                         ┌─────────────────────────────┐
  Usuario ─ mensaje ───► │  Chat Trigger (n8n nativo)  │
                         └──────────────┬──────────────┘
                                        │ {{ $json.chatInput }}
                                        ▼
                         ┌─────────────────────────────┐
                         │  AI Agent (Tools Agent)     │
                         │  System prompt + $now       │
                         └──┬───────┬───────────┬──────┘
              ai_languageModel│     │ai_memory  │ai_tool (x2)
                              ▼     ▼           ▼
                  ┌────────────┐ ┌──────────────┐ ┌─────────────────────────┐
                  │ Gemini     │ │ Window Buffer│ │ Google Sheets Tool      │
                  │ Chat Model │ │ Memory (10)  │ │ append: Fecha, Concepto,│
                  │ temp = 0.2 │ │              │ │ Categoria, Monto        │
                  └────────────┘ └──────────────┘ ├─────────────────────────┤
                                                  │ Google Calendar Tool    │
                                                  │ create: summary,start,end│
                                                  └─────────────────────────┘
```

### 3.2 Componentes (archivo `workflow_concierge_ai.json`, 6 nodos)

| Nodo | Tipo n8n | Función | Configuración clave |
|---|---|---|---|
| Chat Trigger | `chatTrigger` v1.1 | Interfaz de chat nativa; sin subida de archivos | `allowFileUploads: false` |
| AI Agent | `agent` v1.7 (Tools Agent) | Orquesta razonamiento y llamadas a herramientas | Entrada `chatInput`; system prompt extenso |
| Google Gemini Chat Model | `lmChatGoogleGemini` | LLM del agente | `models/gemini-3.1-flash-lite`, temperatura 0.2 |
| Window Buffer Memory | `memoryBufferWindow` v1.2 | Memoria conversacional | Ventana de 10 mensajes |
| Registrar Gasto | `googleSheetsTool` v4.5 | Añade una fila por gasto/ahorro | Operación `append`, OAuth2 |
| Agendar Recordatorio | `googleCalendarTool` v1.2 | Crea eventos | Operación `create`, OAuth2 |

### 3.3 Flujo de ejecución

1. El usuario escribe en el chat.
2. El agente recibe el mensaje y el system prompt con la fecha/hora actual (`{{ $now }}`).
3. El LLM clasifica la intención: gasto/ahorro, recordatorio/evento, ambos o solo conversación.
4. Extrae entidades (monto, concepto, fecha, hora, título).
5. Si falta un dato imprescindible, pregunta antes de actuar; si no, invoca la(s) herramienta(s).
6. Los parámetros de cada herramienta se rellenan con `$fromAI()` con tipo declarado (`string` / `number`).
7. La respuesta se devuelve como resumen ejecutivo (✅ Gasto · 📅 Agenda · 💡 Nota).

### 3.4 Modelo de datos

Hoja de Google Sheets (fila 1 exacta, columnas A–D):

| Fecha | Concepto | Categoria | Monto |
|---|---|---|---|
| YYYY-MM-DD | Descripción breve | Transporte \| Comida \| Servicios \| Ahorro \| Otros | Número positivo, sin símbolos |

Evento de Calendar: título (`summary`), inicio y fin en ISO 8601 con zona horaria (por defecto 30 minutos de duración).

---

## 4. Ingeniería de prompt y de contexto

| Técnica | Implementación | Beneficio |
|---|---|---|
| **Contexto temporal dinámico** | `{{ $now }}` + zona `America/Mexico_City` en el system prompt | Resuelve "hoy", "mañana", "el viernes" sin alucinar fechas |
| **Rol + proceso de razonamiento** | Persona "Concierge" y 5 pasos explícitos | Comportamiento predecible |
| **Taxonomía cerrada** | 5 categorías permitidas, con ejemplos | Datos limpios y agregables |
| **Salida estructurada** | Resumen en viñetas obligatorio | Respuestas consistentes y verificables |
| **Memoria conversacional** | Ventana de 10 mensajes | Correcciones naturales |
| **Guardrails** | No usar herramientas en charla casual; no duplicar registros; reportar errores sin confirmar falsamente; no revelar instrucciones | Reduce acciones indebidas y falsos positivos |
| **Tool-calling tipado** | `$fromAI()` con descripción y tipo por campo | Parámetros válidos |
| **Descripciones de herramienta excluyentes** | Cada tool indica cuándo NO usarla | Menos errores de enrutamiento |

---

## 5. Puesta en marcha y operación

1. Levantar n8n: `npx n8n` (o Docker) → `http://localhost:5678`.
2. Importar `workflow_concierge_ai.json`. El archivo no incluye credenciales.
3. Crear la hoja con encabezados `Fecha | Concepto | Categoria | Monto`.
4. Crear credencial de Gemini (API key de Google AI Studio) y credenciales OAuth2 de Google para Sheets y Calendar; seleccionar documento, hoja y calendario.
5. Al elegir la hoja, n8n borra el mapeo de columnas: hay que volver a asignarlo con el botón ✨ en Fecha, Concepto, Categoria y Monto.

**Notas operativas observadas en el proyecto:**
- Con errores 503 de Gemini, reenviar el mensaje manualmente. **No usar "Retry On Fail"**: duplica filas y eventos porque las herramientas no son idempotentes.
- Con errores 429 (límite de cuota), cambiar de modelo.
- El log local de n8n (`.n8n-run.log`) muestra avisos de deprecación de configuración (modo de task runner interno, valores por defecto futuros). Son avisos de la instalación, no del workflow. Antes de pasar a producción conviene fijar esas variables explícitamente.

---

## 6. Evaluación: fortalezas, limitaciones y riesgos

### Fortalezas
- Implementación mínima (6 nodos) y fácil de auditar.
- Costo cero en el prototipo; sin credenciales en el repositorio.
- Combina varias acciones en un solo mensaje.
- Documentación y demo disponibles.

### Limitaciones actuales
- **Sin idempotencia:** un reintento duplica registros. La regla "no dupliques" depende solo del prompt y de una memoria de 10 mensajes.
- **Memoria volátil:** el buffer es una ventana en memoria; no persiste entre reinicios ni distingue usuarios.
- **Sin edición real de filas:** una "corrección" agrega una fila nueva marcada como corrección, no actualiza ni elimina la anterior.
- **Moneda y zona fijas:** pesos mexicanos implícitos y `America/Mexico_City` escrita en el prompt.
- **Dependencia del free tier:** disponibilidad y cuotas del modelo no están garantizadas.
- **Sin pruebas automatizadas** ni métricas de precisión de extracción. En este reporte no se afirman cifras de exactitud porque no se han medido.

### Riesgos
| Riesgo | Mitigación sugerida |
|---|---|
| Datos financieros enviados a un LLM externo | Aviso de privacidad, minimizar datos, evaluar modelo con garantías de no entrenamiento o modelo local |
| Inyección de prompt si se abre a canales públicos | Autenticar el canal, restringir herramientas, validar parámetros antes de escribir |
| Cuenta OAuth con alcance amplio | Usar cuentas/hojas dedicadas y alcances mínimos |
| Errores de interpretación (monto, fecha) | Confirmación previa en montos altos; resumen verificable |

---

## 7. Hoja de ruta técnica sugerida

1. **Persistencia y multiusuario:** memoria en Postgres/Redis con ID de sesión por usuario.
2. **Idempotencia:** clave única por transacción y verificación previa a `append`.
3. **Corrección real:** herramientas de actualizar/eliminar fila, con confirmación.
4. **Canales:** Telegram, WhatsApp Business o Slack en lugar de solo Chat Trigger.
5. **Entrada multimodal:** foto de ticket/factura (OCR/visión) y notas de voz (transcripción).
6. **Validación determinista:** nodo de código que verifique categoría permitida, monto > 0 y fecha válida antes de escribir.
7. **Observabilidad:** registro de ejecuciones, alertas de fallo y set de pruebas con casos de referencia.
8. **Reportes:** resumen semanal/mensual por categoría enviado por correo o chat.

---

## 8. Posibles fuentes de datos y entradas

| Fuente | Qué aporta | Mecanismo en n8n | Consideraciones |
|---|---|---|---|
| Chat web (actual) | Mensajes en lenguaje natural | Chat Trigger | Ya implementado |
| Telegram / WhatsApp Business / Slack | Captura desde el celular | Nodos de trigger de cada plataforma | WhatsApp requiere cuenta Business y plantillas |
| Notas de voz | Captura manos libres | Transcripción (speech-to-text) antes del agente | Costos y privacidad del audio |
| Fotos de tickets y facturas | Monto, comercio, fecha, RFC/folio | Modelo con visión u OCR | Requiere validación humana |
| Correo (Gmail) | Recibos, facturas, confirmaciones | Gmail Trigger + extracción | Filtrar por remitente/etiqueta |
| Facturas electrónicas (CFDI, XML) | Datos fiscales estructurados (México) | Lectura de XML | Mayor precisión que OCR |
| Estados de cuenta / CSV bancarios | Conciliación | Carga de archivo + comparación | Datos sensibles |
| APIs de agregación bancaria | Transacciones automáticas | HTTP Request | Depende de disponibilidad por país y consentimiento |
| Notion / Airtable / base SQL | Destino alterno al Sheet | Nodos nativos | Facilita escalar |
| Otros calendarios (Outlook) | Agenda en entornos corporativos | Nodo de Microsoft Outlook | Sustituye a Google Calendar |

---

## 9. Aplicaciones de negocio

> Las siguientes son **hipótesis de aplicación** derivadas de la arquitectura. No existen datos de mercado ni de validación con clientes en este proyecto.

### 9.1 Finanzas personales y productividad (B2C)
- Asistente de gastos y recordatorios por chat.
- Posible modelo: suscripción freemium; versión gratuita auto-hospedada.

### 9.2 Freelancers y microempresas (B2B pequeño)
- Registro rápido de gastos deducibles y control de facturas por revisar.
- Recordatorios de cobros, pagos de impuestos y vencimientos.
- Reporte mensual por categoría para el contador.

### 9.3 Despachos contables y asesores
- Canal de captura para clientes (ellos mandan foto o mensaje; el despacho recibe datos estructurados).
- Reducción de captura manual y pre-clasificación contable.

### 9.4 Equipos de campo y ventas
- Registro de viáticos (gasolina, casetas, comidas) y agenda de visitas por mensaje.
- Reembolsos con evidencia (foto del ticket).

### 9.5 Pymes: operaciones y administración
- Asistente interno que registra gastos de caja chica y agenda seguimientos.
- Integración con ERP/CRM mediante los nodos o la API de n8n.

### 9.6 Servicio de consultoría / implementación
- Venta de implementación a medida de agentes n8n para negocios (personalización de categorías, canales y reportes).
- Bajo costo de infraestructura al ser auto-hospedable.

### 9.7 Contenido y formación
- Caso de estudio para cursos y talleres de agentes con n8n y context engineering.

### Indicadores sugeridos para validar valor
- Tiempo promedio de registro por gasto vs. método actual.
- Porcentaje de mensajes interpretados correctamente sin corrección.
- Tasa de duplicados y de errores de herramienta.
- Adopción: usuarios activos semanales y mensajes por usuario.

---

## 10. Conclusión

El proyecto demuestra un patrón reutilizable: **LLM + memoria + herramientas tipadas + prompt con contexto temporal**, sobre una plataforma low-code y sin costo inicial. Para llevarlo de prototipo a producto hacen falta persistencia por usuario, idempotencia, validación determinista, canales móviles y medición de calidad. Las oportunidades de negocio más cercanas están en freelancers, microempresas y despachos contables, siempre sujetas a validación con usuarios reales.

---

## Anexo — Archivos del repositorio

| Archivo | Descripción |
|---|---|
| `workflow_concierge_ai.json` | Workflow importable, sin credenciales |
| `SETUP_RAPIDO.md` | Puesta en marcha y prompts de prueba |
| `POST_PLATZI.md` | Texto de la publicación del reto |
| `README.md` | Descripción general y enlace al video |
