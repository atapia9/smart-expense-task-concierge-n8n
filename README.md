# Smart Expense & Task Concierge

Agente de IA en n8n que, con un solo mensaje en lenguaje natural, registra un gasto en Google Sheets y agenda un recordatorio en Google Calendar. Proyecto para el reto Agent Builders de Platzi. Costo: $0.

> "Pagué 450 pesos de gasolina hoy y recuérdame revisar las facturas el viernes a las 10 am"

**Video demo:** https://youtu.be/LGV56TEeTR8

## Arquitectura

Chat Trigger → AI Agent (Tools Agent) con:
- Google Gemini Chat Model (modelo gratuito)
- Window Buffer Memory (permite correcciones como "la comida fueron 320")
- Google Sheets Tool (añade filas con `Fecha`, `Concepto`, `Categoria`, `Monto`)
- Google Calendar Tool (crea eventos con título, inicio y fin)

El system prompt inyecta `{{ $now }}` para resolver fechas relativas, define las categorías permitidas (Transporte, Comida, Servicios, Ahorro, Otros) y exige un resumen en viñetas.

## Archivos

- `workflow_concierge_ai.json`: workflow importable, sin credenciales.
- `SETUP_RAPIDO.md`: puesta en marcha y mensajes de prueba.
- `POST_PLATZI.md`: texto de la publicación del reto.

## Puesta en marcha

Consulta [SETUP_RAPIDO.md](SETUP_RAPIDO.md). Resumen: `npx n8n`, importa el JSON, crea tus credenciales de Google Gemini, Sheets y Calendar, y elige tu hoja y tu calendario.
