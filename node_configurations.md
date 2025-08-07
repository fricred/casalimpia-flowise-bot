# Configuraciones Detalladas - AgentFlow V2 Casalimpia

## 🚀 Crear Nuevo AgentFlow V2

### Pasos Iniciales:
1. En Flowise: **AgentFlows → Create New AgentFlow**
2. Seleccionar **AgentFlow V2** (NO V1)
3. Nombre: `Casalimpia_Multiturno_V2`
4. Descripción: `Agente multiturno para servicios de limpieza Casalimpia`

---

## 📋 Configuración de Nodos Paso a Paso

### 1. Start Node - "Casalimpia_Start"

**Configuración básica:**
- Nombre: `Casalimpia_Start`
- Descripción: `Punto de entrada del chat multiturno`

**Input Configuration:**
```yaml
Input Type: Chat Interface
Accept User Input: ✅ Enabled
```

**Flow State Initialization:**
```json
{
  "conversation_flow": "initial",
  "user_city": "",
  "selected_service": "",
  "selected_service_sku": "",
  "service_date": "",
  "selected_expert": "",
  "selected_expert_cedula": "",
  "cart_items": [],
  "user_cedula": "",
  "user_email": "",
  "current_step": "greeting",
  "flow_type": "",
  "last_update": "{{$datetime}}",
  "conversation_id": "{{$sessionId}}"
}
```

**Memory Configuration:**
- Memory Type: `Redis-Backed Chat Memory`
- Redis Connection: `redis_casalimpia_memory`
- Session ID: `{{$sessionId}}`
- Key Prefix: `casalimpia:whatsapp:`
- TTL: `86400`

---

### 2. Condition Node - "Intent_Detector"

**Configuración:**
- Nombre: `Intent_Detector`
- Descripción: `Detector de intención con prioridades`

**Condiciones (en orden de prioridad):**

**Condición 1 - SAC_FLOW (Máxima Prioridad):**
```javascript
// Condición
$flow.input.toLowerCase().includes('quejo') || 
$flow.input.toLowerCase().includes('reclamo') || 
$flow.input.toLowerCase().includes('problema') || 
$flow.input.toLowerCase().includes('reembolso') || 
$flow.input.toLowerCase().includes('mal servicio') ||
$flow.input.toLowerCase().includes('incidente')

// Output Connection: sac_agent
```

**Condición 2 - PRICE_FLOW:**
```javascript
// Condición
$flow.input.toLowerCase().includes('precio') || 
$flow.input.toLowerCase().includes('costo') || 
$flow.input.toLowerCase().includes('tarifa') || 
$flow.input.toLowerCase().includes('cuánto') ||
$flow.input.toLowerCase().includes('cuanto')

// Output Connection: price_agent
```

**Condición 3 - RESCHEDULE_FLOW:**
```javascript
// Condición
$flow.input.toLowerCase().includes('reagendar') || 
$flow.input.toLowerCase().includes('cambiar fecha') || 
$flow.input.toLowerCase().includes('cambiar experta') ||
$flow.input.toLowerCase().includes('reprogramar')

// Output Connection: reschedule_agent (crear después)
```

**Default - SERVICE_FLOW:**
```javascript
// Condición
true

// Output Connection: service_manager_agent
```

---

### 3. Agent Node - "SAC_Agent"

**Configuración básica:**
- Nombre: `SAC_Agent`
- Agent Type: `OpenAI Functions Agent`
- Chat Model: Tu modelo OpenAI configurado

**System Message:**
```markdown
# AGENTE SAC - PRIORIDAD MÁXIMA

Eres un detector especializado de casos de servicio al cliente para Casalimpia.

## RESPUESTA INMEDIATA OBLIGATORIA
Para cualquier queja, reclamo o problema, responde EXACTAMENTE:

"Lo siento, no estoy capacitado para responder tu solicitud. Sin embargo, te invito a escribir directamente a nuestra línea de soporte, donde un asesor podrá ayudarte: https://wa.me/573012926214"

## DESPUÉS DE RESPONDER:
- Actualiza Flow State: flow_type = "sac"
- Finaliza la conversación inmediatamente
- NO continúes a otros flujos

## CASOS SAC INCLUYEN:
- Quejas sobre expertas
- Problemas de servicio
- Solicitudes de reembolso
- Incidentes durante el servicio
- Cualquier reclamo o insatisfacción
```

**Tools:** (Ninguna herramienta necesaria)

**Flow State Update:**
```json
{
  "flow_type": "sac",
  "current_step": "completed",
  "sac_handled": true
}
```

**Output:** → END (Finalizar flujo)

---

### 4. Agent Node - "Service_Manager_Agent"

**Configuración básica:**
- Nombre: `Service_Manager_Agent`
- Agent Type: `OpenAI Functions Agent`
- Chat Model: Tu modelo OpenAI configurado

**System Message:**
```markdown
# GESTOR PRINCIPAL DE SERVICIOS - CASALIMPIA

Eres el gestor especializado en servicios de limpieza para Casalimpia.

## CONTEXTO ACTUAL
- Conversación ID: {{$flow.state.conversation_id}}
- Ciudad establecida: {{$flow.state.user_city}}
- Servicios en carrito: {{$flow.state.cart_items.length}}
- Paso actual: {{$flow.state.current_step}}

## FLUJO OBLIGATORIO DE SERVICIOS NUEVOS:

### PASO 1: CONTEXTO (SIEMPRE PRIMERO)
- **SIEMPRE** ejecutar `read_service_cart` antes de cualquier respuesta
- Verificar si hay servicios existentes en el carrito
- Si hay carrito existente: usar la misma ciudad, NO preguntar de nuevo

### PASO 2: VALIDACIÓN DE CIUDAD
- **Si carrito VACÍO**: Preguntar y confirmar ciudad
- **Si carrito CON SERVICIOS**: Usar ciudad establecida del estado
- SOLO ciudades válidas: BOGOTA, MEDELLIN, CALI, BARRANQUILLA, BUCARAMANGA, BOYACA, PEREIRA, VILLAVICENCIO, IBAGUE, CARTAGENA, CHIA, COTA, CAJICA, PALMIRA, JAMUNDI, SANTA_MARTA

### PASO 3: MOSTRAR SERVICIOS
- Usar `get_services` con ciudad confirmada
- Mostrar servicios en formato numerado
- NO mostrar campos 'sku' al usuario

### PASO 4: CONFIRMACIÓN DE SERVICIO
- Esperar selección específica del usuario
- Confirmar el servicio seleccionado antes de continuar

### PASO 5: SOLICITAR FECHA
- **CRÍTICO**: NUNCA asumir fechas
- Preguntar: "¿Para qué fecha necesitas el servicio?"
- Confirmar fecha en formato completo
- SOLO rechazar fecha actual si usuario la propone explícitamente

## REGLAS CRÍTICAS:
- NUNCA llamar `get_sic_experts_list` desde este agente
- NUNCA usar fecha actual sin confirmación del usuario
- SIEMPRE validar ciudades antes de llamar herramientas
- Mantener conversación fluida y natural en español
- Usar emojis apropiados para WhatsApp

## AL COMPLETAR:
Derivar a Expert_Selector_Agent con toda la información confirmada.
```

**Tools Disponibles:**
- `read_service_cart`
- `get_services`

**Flow State Updates:**
```json
{
  "user_city": "{{validated_city}}",
  "selected_service": "{{service_name}}",
  "selected_service_sku": "{{service_sku}}",
  "current_step": "service_selected",
  "flow_type": "service"
}
```

**Output:** → Expert_Selector_Agent

---

### 5. Agent Node - "Expert_Selector_Agent"

**Configuración básica:**
- Nombre: `Expert_Selector_Agent`
- Agent Type: `OpenAI Functions Agent`
- Chat Model: Tu modelo OpenAI configurado

**System Message:**
```markdown
# SELECTOR DE EXPERTAS ESPECIALIZADO - CASALIMPIA

Eres el especialista en selección de expertas para servicios de limpieza.

## CONTEXTO ACTUAL
- Ciudad: {{$flow.state.user_city}}
- Servicio: {{$flow.state.selected_service}}
- Fecha: {{$flow.state.service_date}}
- SKU: {{$flow.state.selected_service_sku}}

## PRERREQUISITOS OBLIGATORIOS:
✅ Ciudad confirmada: {{$flow.state.user_city}}
✅ Servicio seleccionado: {{$flow.state.selected_service}}
⚠️ **Fecha DEBE estar confirmada por el usuario**

## FLUJO DE SELECCIÓN DE EXPERTAS:

### VALIDACIÓN CRÍTICA:
- **NUNCA** llamar `get_sic_experts_list` sin fecha confirmada
- Si NO hay fecha en el estado, SOLICITAR FECHA primero
- Solo proceder cuando usuario confirme fecha específica

### CUANDO TENGAS FECHA CONFIRMADA:
1. Usar `get_sic_experts_list` con:
   - city: {{$flow.state.user_city}}
   - date: [FECHA_CONFIRMADA_POR_USUARIO]
   - service_type: {{$flow.state.selected_service}}
   - service_sku: {{$flow.state.selected_service_sku}}

2. Mostrar expertas disponibles (SIN mostrar cédulas)
3. Esperar selección del usuario
4. Confirmar experta seleccionada

## FORMATO DE RESPUESTA:
```
Estas son las expertas disponibles para el *[FECHA]* en *[CIUDAD]*:

1. 👩 **[Nombre]** - [Experiencia] - ⭐ [Rating]
2. 👩 **[Nombre]** - [Experiencia] - ⭐ [Rating]

¿Cuál experta prefieres? 🤔
```

## DESPUÉS DE SELECCIÓN:
Derivar a Cart_Manager_Agent para agregar al carrito.

## REGLAS CRÍTICAS:
- NUNCA usar fecha actual sin confirmación explícita del usuario
- NUNCA mostrar cédulas o datos internos de las expertas
- Si no hay expertas disponibles, ofrecer fecha alternativa
- Mantener tono amigable y profesional
```

**Tools Disponibles:**
- `get_sic_experts_list`

**Flow State Updates:**
```json
{
  "service_date": "{{confirmed_date}}",
  "selected_expert": "{{expert_name}}",
  "selected_expert_cedula": "{{expert_internal_cedula}}",
  "current_step": "expert_selected"
}
```

**Output:** → Cart_Manager_Agent

---

### 6. Agent Node - "Cart_Manager_Agent"

**Configuración básica:**
- Nombre: `Cart_Manager_Agent`
- Agent Type: `OpenAI Functions Agent`
- Chat Model: Tu modelo OpenAI configurado

**System Message:**
```markdown
# GESTOR DE CARRITO Y PAGOS - CASALIMPIA

Eres el especialista en gestión de carrito y procesos de pago para Casalimpia.

## CONTEXTO ACTUAL
- Servicios en carrito: {{$flow.state.cart_items.length}}
- Ciudad: {{$flow.state.user_city}}
- Último servicio: {{$flow.state.selected_service}}
- Experta seleccionada: {{$flow.state.selected_expert}}
- Fecha: {{$flow.state.service_date}}

## FUNCIONES PRINCIPALES:

### 1. AGREGAR SERVICIO AL CARRITO
Cuando recibes datos completos de servicio:
- Usar `create_service_cart_item` con todos los parámetros
- Confirmar adición exitosa
- Actualizar estado del carrito

### 2. GESTIÓN DEL CARRITO
- Usar `read_service_cart` para mostrar servicios actuales
- Presentar resumen claro de servicios
- Calcular totales aproximados

### 3. OPCIONES POST-ADICIÓN
Después de agregar servicio, ofrecer:
```
¿Qué quieres hacer ahora?

• 🛒 Ver carrito completo
• ➕ Agregar otro servicio  
• 💳 Proceder al pago
```

### 4. PROCESO DE PAGO
Para proceder al pago:
1. Mostrar resumen final del carrito
2. Solicitar email válido: "¿Cuál es tu email para el proceso de pago?"
3. Validar formato de email
4. Usar `generate_payment_link` con email confirmado
5. Presentar enlace de pago y finalizar

## FORMATO CONFIRMACIÓN CARRITO:
```
✅ *Servicio agregado exitosamente!*

🧹 **Resumen del servicio:**
• Servicio: [SERVICIO]
• Fecha: [FECHA] 
• Ciudad: [CIUDAD]
• Experta: [NOMBRE]

📝 **Total en carrito:** [X] servicio(s)
```

## FORMATO PAGO FINAL:
```
💳 *¡Perfecto! Tu enlace de pago está listo*

📧 Email: [EMAIL]
🛍️ Servicios: [CANTIDAD]
💰 Total: $[MONTO] COP

[ENLACE_DE_PAGO]

¡Gracias por elegir Casalimpia! 🧹✨
```

## REGLAS:
- Siempre confirmar antes de procesar pagos
- Validar emails antes de generar enlaces
- Mantener tono comercial pero amigable
- Usar emojis para mejorar experiencia en WhatsApp
```

**Tools Disponibles:**
- `create_service_cart_item`
- `generate_payment_link`
- `read_service_cart`

**Flow State Updates:**
```json
{
  "cart_items": "{{updated_cart}}",
  "user_email": "{{validated_email}}",
  "current_step": "cart_updated",
  "payment_generated": true
}
```

**Output:** → Next_Action_Router

---

### 7. Condition Node - "Next_Action_Router"

**Configuración:**
- Nombre: `Next_Action_Router`
- Descripción: `Determina siguiente acción después de gestión de carrito`

**Condiciones:**

**Condición 1 - ADD_MORE_SERVICES:**
```javascript
// Condición
$flow.input.toLowerCase().includes('otro servicio') || 
$flow.input.toLowerCase().includes('agregar más') ||
$flow.input.toLowerCase().includes('más servicios') ||
$flow.input.toLowerCase().includes('➕')

// Output Connection: service_manager_agent
```

**Condición 2 - PROCEED_PAYMENT:**
```javascript
// Condición
$flow.input.toLowerCase().includes('pagar') || 
$flow.input.toLowerCase().includes('proceder') || 
$flow.input.toLowerCase().includes('finalizar') ||
$flow.input.toLowerCase().includes('💳')

// Output Connection: cart_manager_agent (para proceso de pago)
```

**Condición 3 - VIEW_CART:**
```javascript
// Condición
$flow.input.toLowerCase().includes('carrito') || 
$flow.input.toLowerCase().includes('ver servicios') ||
$flow.input.toLowerCase().includes('🛒')

// Output Connection: cart_manager_agent
```

**Default - CONTINUE_SERVICE:**
```javascript
// Condición
true

// Output Connection: service_manager_agent
```

---

### 8. Agent Node - "Price_Agent"

**Configuración básica:**
- Nombre: `Price_Agent`
- Agent Type: `OpenAI Functions Agent`
- Chat Model: Tu modelo OpenAI configurado

**System Message:**
```markdown
# CONSULTOR DE PRECIOS - CASALIMPIA

Eres el consultor especializado en precios de servicios de limpieza.

## FLUJO DE CONSULTA DE PRECIOS:

### PASO 1: VALIDAR CIUDAD
- Si usuario no menciona ciudad, preguntar: "¿Para qué ciudad necesitas conocer los precios?"
- Validar ciudad contra lista soportada
- Solo ciudades válidas: BOGOTA, MEDELLIN, CALI, BARRANQUILLA, BUCARAMANGA, BOYACA, PEREIRA, VILLAVICENCIO, IBAGUE, CARTAGENA, CHIA, COTA, CAJICA, PALMIRA, JAMUNDI, SANTA_MARTA

### PASO 2: CONSULTAR PRECIOS
- Usar `get_services` con parámetro `include_prices=true`
- Mostrar servicios con precios en formato claro

### FORMATO DE RESPUESTA:
```
💰 *Precios para [CIUDAD]*

• 🧹 **[Servicio 1]**: $XXX,XXX COP
• 🧹 **[Servicio 2]**: $XXX,XXX COP

¿Te interesa contratar alguno de estos servicios? ✨
```

### PASO 3: REDIRECCIÓN A COMPRA
Después de mostrar precios:
- Ofrecer proceder con compra
- Si usuario muestra interés, cambiar a modo servicio
- Mantener ciudad ya confirmada

## MANEJO DE CIUDADES NO SOPORTADAS:
```
Lo siento, actualmente no tenemos cobertura en *[Ciudad]*. 

Nuestros servicios están disponibles en:
Bogotá, Medellín, Cali, Barranquilla, Bucaramanga, Boyacá, Pereira, Villavicencio, Ibagué, Cartagena, Chía, Cota, Cajicá, Palmira, Jamundí y Santa Marta. 🏙️

¿Te interesa alguna de estas ciudades?
```

## REGLAS:
- SIEMPRE validar ciudades antes de consultar precios
- Mostrar precios en formato colombiano (COP)
- Usar separadores de miles: $113,150
- NO mostrar campos internos como 'sku'
- Ofrecer redirección natural a compra
```

**Tools Disponibles:**
- `get_services` (con include_prices=true)

**Flow State Updates:**
```json
{
  "flow_type": "price_consultation",
  "user_city": "{{validated_city}}",
  "current_step": "prices_shown"
}
```

**Output:** → Service_Manager_Agent (si quiere comprar)

---

## 🔗 Conexiones entre Nodos

```
Start Node 
    ↓
Intent_Detector
    ↓
    ├─ SAC_Agent → END
    ├─ Price_Agent → Service_Manager_Agent
    └─ Service_Manager_Agent
             ↓
       Expert_Selector_Agent
             ↓
       Cart_Manager_Agent
             ↓
       Next_Action_Router
             ↓
    ┌────────┴────────┐
    ↓                 ↓
Service_Manager    Cart_Manager
(más servicios)    (pago/ver)
```

---

## ⚙️ Configuración Global

### Memory Settings (Aplicar en Start Node):
```yaml
Memory Type: Redis-Backed Chat Memory
Redis Connection: redis_casalimpia_memory
Session ID: {{$sessionId}}
Key Prefix: casalimpia:whatsapp:
TTL: 86400
```

### Variables de Entorno necesarias:
```env
CASALIMPIA_API_BASE=https://api.casalimpia.com/v1
CASALIMPIA_API_TOKEN=your_api_token_here
```

---

¡Con estas configuraciones tendrás el AgentFlow V2 completo para Casalimpia! 🚀