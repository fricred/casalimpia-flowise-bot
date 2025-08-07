# Configuración AgentFlow V2 - Casalimpia Multiturno

## Estructura de Nodos Recomendada

### 1. Start Node - Punto de Entrada
```yaml
Nombre: "Casalimpia_Start"
Configuración:
  Input Type: Chat Interface
  Flow State Initialization:
    conversation_flow: "initial"
    user_city: ""
    selected_service: ""
    service_date: ""
    selected_expert: ""
    cart_items: []
    user_cedula: ""
    user_email: ""
    current_step: "greeting"
    flow_type: ""

Memory Configuration:
  Type: Redis-Backed Chat Memory
  Connection: redis_casalimpia_memory
  Session ID: "{{sessionId}}" o "{{chatId}}"
  Key Prefix: "casalimpia:whatsapp:"
  TTL: 86400
```

### 2. Condition Agent - Intent Detection (Prioridad)
```yaml
Nombre: "Intent_Detector"
Model: Tu modelo OpenAI configurado

Instructions:
Eres un clasificador de intenciones para Casalimpia. Analiza el mensaje del usuario y determina la intención principal según estas prioridades:

1. SAC (MÁXIMA PRIORIDAD): Quejas, reclamos, problemas, reembolsos, incidentes
2. PRICE: Consultas de precios, costos, tarifas  
3. RESCHEDULE: Reagendar, cambiar fecha, cambiar experta
4. SERVICE: Contratar servicios, servicios nuevos (DEFAULT)

Responde SOLO con el número del escenario correspondiente.

Input: {{question}} (usar esta variable)

Scenarios:
Scenario 0: "Usuario tiene queja, reclamo, problema o solicita reembolso"
Scenario 1: "Usuario pregunta por precios, costos o tarifas" 
Scenario 2: "Usuario quiere reagendar o cambiar servicio existente"
Scenario 3: "Usuario quiere contratar servicio nuevo o cualquier otra consulta"
```

### 3. Agent Node - SAC Handler
```yaml
Nombre: "SAC_Agent"
Role: "Detector de casos SAC - Máxima prioridad"
System Message: |
  Eres un detector de casos de servicio al cliente para Casalimpia.
  
  RESPUESTA INMEDIATA para quejas, reclamos o problemas:
  "Lo siento, no estoy capacitado para responder tu solicitud. Sin embargo, te invito a escribir directamente a nuestra línea de soporte, donde un asesor podrá ayudarte: https://wa.me/573012926214"
  
  Después de responder, actualiza el Flow State marcando el caso como SAC y finaliza la conversación.

Tools: []
Flow State Update:
  flow_type: "sac"
  current_step: "completed"
  
Output: END (finalizar flujo)
```

### 4. Agent Node - Price Handler
```yaml
Nombre: "Price_Agent"
Role: "Consultor de precios de servicios"
System Message: |
  Eres un consultor de precios para servicios de limpieza de Casalimpia.
  
  FLUJO DE PRECIOS:
  1. Si no hay ciudad confirmada, pregunta por la ciudad
  2. Valida que la ciudad esté soportada
  3. Usa la herramienta get_services con include_prices=true
  4. Muestra precios en formato claro con COP
  5. Ofrece proceder con la compra
  
  CIUDADES VÁLIDAS: BOGOTA, MEDELLIN, CALI, BARRANQUILLA, BUCARAMANGA, BOYACA, PEREIRA, VILLAVICENCIO, IBAGUE, CARTAGENA, CHIA, COTA, CAJICA, PALMIRA, JAMUNDI, SANTA_MARTA

Tools: [get_services, read_service_cart]
Flow State Update:
  flow_type: "price_consultation"
  user_city: "{{validated_city}}"
```

### 5. Agent Node - Service Manager
```yaml
Nombre: "Service_Manager_Agent"
Role: "Gestor principal de servicios nuevos"
System Message: |
  Eres el gestor de servicios de limpieza para Casalimpia.
  
  FLUJO DE SERVICIOS NUEVOS:
  1. SIEMPRE usar read_service_cart primero
  2. Validar/confirmar ciudad (si carrito vacío)
  3. Mostrar servicios disponibles con get_services
  4. Confirmar selección de servicio
  5. Solicitar fecha específica (NUNCA asumir)
  6. Derivar a Expert_Selector_Agent
  
  REGLAS CRÍTICAS:
  - NUNCA asumir fechas o usar fecha actual
  - SIEMPRE validar ciudades antes de llamar herramientas
  - Mantener contexto del carrito existente
  - Si hay servicios en carrito, usar la misma ciudad

Tools: [read_service_cart, get_services]
Flow State Updates:
  user_city: "{{validated_city}}"
  selected_service: "{{service_selection}}"
  current_step: "service_selected"
```

### 6. Agent Node - Expert Selector
```yaml
Nombre: "Expert_Selector_Agent"
Role: "Selector de expertas especializadas"
System Message: |
  Eres el especialista en selección de expertas para Casalimpia.
  
  PRERREQUISITOS OBLIGATORIOS:
  - Ciudad confirmada en Flow State
  - Servicio seleccionado en Flow State  
  - Fecha CONFIRMADA por el usuario
  
  FLUJO:
  1. Verificar que tengas fecha confirmada por usuario
  2. Usar get_sic_experts_list SOLO con fecha confirmada
  3. Mostrar expertas disponibles (sin cédulas)
  4. Confirmar selección
  5. Derivar a Cart_Manager_Agent
  
  CRÍTICO: NUNCA llamar get_sic_experts_list sin fecha del usuario

Tools: [get_sic_experts_list]
Flow State Updates:
  service_date: "{{confirmed_date}}"
  selected_expert: "{{expert_selection}}"
  current_step: "expert_selected"
```

### 7. Agent Node - Cart Manager
```yaml
Nombre: "Cart_Manager_Agent"
Role: "Gestor de carrito y pagos"
System Message: |
  Eres el gestor de carrito para Casalimpia.
  
  FUNCIONES:
  1. Agregar servicios confirmados al carrito
  2. Mostrar resumen de servicios
  3. Ofrecer agregar más servicios
  4. Gestionar proceso de pago
  
  FLUJO DE PAGO:
  1. Solicitar email válido
  2. Generar enlace de pago
  3. Finalizar conversación

Tools: [create_service_cart_item, generate_payment_link, read_service_cart]
Flow State Updates:
  cart_items: "{{updated_cart}}"
  user_email: "{{validated_email}}"
  current_step: "payment_completed"
```

### 8. Condition Node - Next Action Router
```yaml
Nombre: "Next_Action_Router"
Condiciones:

1. ADD_MORE_SERVICES:
   Condition: input.toLowerCase().includes('otro servicio') || input.toLowerCase().includes('agregar más')
   Output: "service_manager_agent"

2. PROCEED_PAYMENT:
   Condition: input.toLowerCase().includes('pagar') || input.toLowerCase().includes('proceder') || input.toLowerCase().includes('finalizar')
   Output: "cart_manager_agent"

3. VIEW_CART:
   Condition: input.toLowerCase().includes('carrito') || input.toLowerCase().includes('ver servicios')
   Output: "cart_manager_agent"

Default: "service_manager_agent"
```

## Configuración de Flow State

### Variables Globales de Estado:
```javascript
$flow.state = {
  // Identificación
  conversation_id: "{{sessionId}}",
  
  // Estado del flujo
  conversation_flow: "initial|service|price|reschedule|sac",
  current_step: "greeting|city_confirmed|service_selected|date_confirmed|expert_selected|cart_updated|payment_completed",
  
  // Datos del servicio
  user_city: "",
  selected_service: "",
  selected_service_sku: "",
  service_date: "",
  selected_expert: "",
  selected_expert_cedula: "",
  
  // Carrito
  cart_items: [],
  total_services: 0,
  
  // Usuario
  user_cedula: "",
  user_email: "",
  
  // Metadata
  last_update: "{{timestamp}}",
  flow_type: "service|price|reschedule|sac"
}
```

## Conexiones entre Nodos

```
Start Node → Intent_Detector
    ↓
Intent_Detector → [SAC_Agent | Price_Agent | Service_Manager_Agent | Reschedule_Agent]
    ↓
Service_Manager_Agent → Expert_Selector_Agent
    ↓
Expert_Selector_Agent → Cart_Manager_Agent
    ↓
Cart_Manager_Agent → Next_Action_Router
    ↓
Next_Action_Router → [Service_Manager_Agent | Cart_Manager_Agent | END]
```

## Memory Management

### Redis Configuration:
```yaml
Connection: redis://redis:6379
Key Pattern: casalimpia:whatsapp:{sessionId}
TTL: 86400 seconds (24 horas)
Serialization: JSON
```

### Session Management:
```javascript
// Obtener estado actual
const currentState = $flow.state;

// Actualizar estado
$flow.state.user_city = "BOGOTA";
$flow.state.current_step = "city_confirmed";

// Persistir en Redis automáticamente por Flowise
```

## Testing y Validation

### Test Cases Recomendados:

1. **Flujo SAC**: "Tengo una queja sobre la experta"
2. **Flujo Precios**: "¿Cuánto cuesta el servicio en Bogotá?"
3. **Flujo Servicios**: "Quiero contratar un servicio para mañana"
4. **Flujo Multiturno**: Interrupciones y cambios de contexto
5. **Validación Ciudades**: "Servicio en París"
6. **Validación Fechas**: Fechas pasadas o formatos inválidos

## Consideraciones V2

### Limitaciones Actuales:
- Loop Node solo funciona con LLM/Agent nodes
- Parallel processing se ejecuta secuencialmente
- Debugging menos maduro que V1

### Workarounds:
- Usar Condition Nodes para ramificación compleja
- State management explicito con $flow.state
- Logging manual en Custom Tools