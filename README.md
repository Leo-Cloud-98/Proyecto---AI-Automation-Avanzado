# M1 – Agente base: Asistente de Onboarding de Inversores (Demo Valores, ficticio)

Checkpoint 1 del curso AI Automation Avanzado.

**Flujo:** Chat Trigger → AI Agent (Tools Agent) → Slack (log de observabilidad)

- AI Agent en modo Tools Agent (es el modo por defecto, por eso no figura en el JSON). Iteraciones máximas: 6.
- System Prompt modular: Rol → Ámbito → Objetivo → Reglas y Escalamiento, con acciones prohibidas explícitas.
- Tool acoplada: Google Sheets "consultar_manual_broker" (manual ficticio del broker), con description de negocio.
- Observabilidad: mensaje de Slack con consulta, respuesta, herramientas usadas y su resultado.
- Modelo: Google Gemini (por disponibilidad de plan gratuito). El nodo es intercambiable por OpenAI o Anthropic.
