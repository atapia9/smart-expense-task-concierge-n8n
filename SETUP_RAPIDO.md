# Setup rápido
1. **Levantar n8n:** `npx n8n` (o `docker run -it --rm -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n`) → http://localhost:5678
2. **Importar:** Workflows → ⋯ → *Import from file* → `workflow_concierge_ai.json`.
3. **Google Sheet:** fila 1 exacta (A–D): `Fecha | Concepto | Categoria | Monto`
4. **API Key gratis:** aistudio.google.com/apikey → en el nodo *Google Gemini Chat Model* → Credential → *Create new* → pega la key.
5. **Sheets y Calendar:** en cada Tool elige tu credencial OAuth2 de Google y selecciona tu Documento/Hoja y tu Calendario.
5b. Al elegir tu hoja, n8n borra el mapeo: en *Registrar Gasto* pulsa ✨ junto a Fecha, Concepto, Categoria y Monto. Si Gemini da 503, reenvía el mensaje (no uses Retry On Fail: duplica filas y eventos); si da 429 cambia de modelo.
6. Guarda, pulsa **Open chat** (o *Test workflow*) y prueba.

**Prompts para el demo de 45 s**
- 💸 Gasto: `Pagué 450 pesos de gasolina hoy`
- 📅 Agenda: `Recuérdame revisar las facturas el viernes a las 10 am`
- 🧠 Combinado + memoria: `Gasté 280 en comida y agenda llamar al contador mañana a las 4 pm` → luego: `Corrección: la comida fueron 320 y la llamada a las 5 pm`
