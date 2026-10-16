🚀 **Smart Expense & Task Concierge — un agente en n8n que convierte una frase en un gasto registrado y un evento agendado**

**🎯 Problema y usuario**
Freelancers, emprendedores y profesionales ocupados anotan gastos "después" y olvidan pendientes porque saltar entre Sheets, Calendar y notas rompe el flujo. Resultado: finanzas desordenadas y recordatorios perdidos. Mi agente reduce eso a **un solo mensaje en lenguaje natural**:

> "Pagué 450 pesos de gasolina hoy y recuérdame revisar las facturas el viernes a las 10 am"

El agente entiende la intención, extrae monto, categoría y fecha, y ejecuta una o ambas acciones. Costo total: **$0** (Gemini free tier + Google Sheets/Calendar + n8n self-hosted).

**🧠 Arquitectura de Prompt / Context Engineering**
- **Contexto temporal dinámico:** el system prompt inyecta `{{ $now }}` y la zona horaria, así "el viernes" o "mañana" se resuelven con fechas ISO exactas, sin alucinar.
- **Rol + proceso de razonamiento explícito:** detectar intención → extraer entidades → elegir herramienta(s) → preguntar solo si falta un dato crítico.
- **Taxonomía cerrada:** categorías permitidas (Transporte, Comida, Servicios, Ahorro, Otros) para datos limpios y analizables.
- **Salida estructurada:** resumen ejecutivo en viñetas (✅ Gasto · 📅 Agenda · 💡 Nota).
- **Memoria conversacional:** Window Buffer Memory (10 mensajes) permite correcciones naturales: "la comida fueron 320, no 280".
- **Guardrails:** no ejecuta herramientas en charla casual, no duplica registros, pide aclaración antes de adivinar.
- **Tool-calling con `$fromAI()`:** el modelo rellena los parámetros de cada herramienta con tipos definidos (string/number).

**🛠️ Herramientas usadas**
n8n · Nodo **AI Agent (Tools Agent)** · **Google Gemini 3.1 Flash Lite** (gratis) · **Window Buffer Memory** · Tool **Google Sheets** (append de gastos) · Tool **Google Calendar** (create de eventos) · Chat Trigger nativo. Sin credenciales expuestas en el JSON.

**📹 Demo (1:46):** https://youtu.be/LGV56TEeTR8 — gasto simple → recordatorio → mensaje combinado + corrección usando memoria.
🔗 Workflow JSON importable: https://github.com/atapia9/smart-expense-task-concierge-n8n

**🙌 ¿Te sirvió?** Si te gustó la idea, **déjame tu voto y un comentario** con qué otra herramienta le conectarías (Gmail, Notion, WhatsApp…). ¡Gracias por apoyar! 💚

#AgentBuilders #n8n #Platzi #IA #Automatización
