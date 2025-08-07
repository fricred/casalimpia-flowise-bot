# Casalimpia Flowise Bot

Chatbot multiagente para Casalimpia desarrollado con Flowise AgentFlow usando patrón Supervisor.

## 🚀 Características

- **Supervisor Pattern**: Enrutamiento inteligente usando LLM con structured output
- **5 Agentes Especializados**: Greeting, SAC, Price, Reschedule, Service
- **Multiturno**: Memoria persistente a través de conversaciones
- **WhatsApp Format**: Respuestas optimizadas con emojis
- **Tools Integradas**: 7 herramientas personalizadas para APIs de Casalimpia

## 📁 Archivos Principales

- `Casalimpia Supervision.json` - **Configuración principal** (usar esta)
- `Casalimpia Multiturno Agents.json` - Versión anterior con ConditionAgent
- Tools personalizadas: `*-CustomTool.json`

## 🏗️ Arquitectura

```
Start → Supervisor (LLM) → [Greeting|SAC|Price|Reschedule|Service]
```

## 🛠️ Setup

1. Importar `Casalimpia Supervision.json` en Flowise
2. Importar las custom tools necesarias
3. Configurar variables de entorno en `.env`
4. ¡Listo para chatear!

## 🎯 Funcionalidades

- **Saludos**: Menú de opciones interactivo
- **Consulta Precios**: Por ciudad con 16 ubicaciones soportadas
- **Reagendamiento**: Cambio de fecha/experta de servicios existentes
- **Contratación**: Proceso completo de compra con carrito y pago
- **SAC**: Derivación automática a soporte humano

## 🔧 Tecnologías

- Flowise AgentFlow V2
- OpenAI GPT-4o-mini
- Custom Tools para integración API
- WhatsApp Business API compatible

---

Desarrollado con ❤️ para Casalimpia 🧹✨
