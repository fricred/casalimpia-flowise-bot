# Start Node - Configuración Detallada con Persist State

## 🚀 Start Node Configuration - "Casalimpia_Start"

### Configuración Básica
- **Node Type**: Start Node
- **Name**: `Casalimpia_Start`
- **Description**: `Punto de entrada con estado persistente para chat multiturno`

---

## 📝 Input Configuration

### Input Settings:
```yaml
Input Type: Chat Interface
Accept User Input: ✅ Enabled
Show Form: ❌ Disabled (para chat directo)
```

---

## 🗂️ Flow State Configuration

### ⚡ **CRITICAL: Persist State Settings**

En el Start Node, busca la sección **"Flow State"** o **"State Management"**:

**Enable Persist State:** ✅ **ACTIVAR**

**Initial State (JSON):**
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
  "total_services": 0,
  "last_update": "",
  "conversation_id": "{{$sessionId}}",
  "session_started": "{{$datetime}}",
  "last_interaction": "{{$datetime}}"
}
```

---

## 💾 Memory Configuration

### Redis Memory Setup:
```yaml
Memory Type: Redis-Backed Chat Memory
Connection: redis_casalimpia_memory (tu credential creado)
```

### Session Configuration:
```yaml
Session ID: {{$sessionId}}
# Alternativamente puedes usar: {{$chatId}}

Key Prefix: casalimpia:whatsapp:
TTL (seconds): 86400  # 24 horas
```

### Memory Settings Avanzados:
```yaml
Return Messages: 10  # Últimos 10 mensajes en contexto
Memory Key: chat_history
```

---

## 🎯 Variables de Sistema Disponibles

En AgentFlow V2 tienes acceso a:
```javascript
$sessionId    // ID único de sesión
$chatId       // ID de chat (alternativo)
$chatflowId   // ID del flow actual
$input        // Mensaje actual del usuario
$datetime     // Timestamp actual
$flow.state   // Estado persistente del flow
```

---

## 🔧 System Message (Opcional en Start Node)

Si el Start Node permite System Message:
```markdown
# ASISTENTE VIRTUAL CASALIMPIA - INICIALIZACIÓN

Sistema iniciado para conversación multiturno.
- Session ID: {{$sessionId}}
- Estado inicial: {{$flow.state}}
- Memoria Redis: Activa con TTL 24h

Listo para procesar intenciones y mantener contexto conversacional.
```

---

## ⚙️ Advanced Settings

### State Persistence Options:
```yaml
Persist State: ✅ Enabled
State Storage: Redis (desde Memory config)
State TTL: 86400 seconds
Auto-save State: ✅ Enabled
State Compression: ✅ Enabled (opcional, para optimizar)
```

### Error Handling:
```yaml
On State Load Error: Initialize new state
On Memory Error: Continue without history
Max Retries: 3
```

---

## 🔗 Connection Output

El Start Node debe conectarse directamente a:
**Output:** → `Intent_Detector` (Condition Node)

---

## 📋 Checklist de Verificación

### ✅ Configuraciones Obligatorias:
- [ ] Input Type: Chat Interface habilitado
- [ ] **Persist State: ACTIVADO**
- [ ] Initial State JSON configurado
- [ ] Redis Memory configurado con credencial
- [ ] Session ID: {{$sessionId}}
- [ ] TTL: 86400 (24h)
- [ ] Key Prefix: casalimpia:whatsapp:
- [ ] Conectado a Intent_Detector

### 🧪 Pruebas de Estado:

**Test 1 - Estado Inicial:**
```
Input: "Hola"
Expected State: 
{
  "current_step": "greeting",
  "conversation_flow": "initial",
  "cart_items": []
}
```

**Test 2 - Persistencia:**
```
Session 1: "Quiero servicio en Bogotá"
Session 2: (misma sesión) "¿Qué tengo en el carrito?"
Expected: Debe recordar Bogotá del mensaje anterior
```

---

## 🔍 Debugging Tips

### Verificar Estado en Runtime:
En cualquier Agent Node posterior, puedes acceder:
```javascript
// Ver estado completo
console.log("Current State:", JSON.stringify($flow.state, null, 2));

// Verificar campos específicos
console.log("User City:", $flow.state.user_city);
console.log("Cart Items:", $flow.state.cart_items.length);
```

### Variables de Debug útiles:
```javascript
$flow.sessionId     // Para identificar sesión única
$flow.state         // Estado completo actual
$flow.input         // Input del usuario actual
```

---

## ⚠️ Notas Importantes

### 1. **Redis Connection:**
- Asegúrate de que la credential `redis_casalimpia_memory` esté configurada correctamente
- Host: `redis` (nombre del servicio Docker)
- Port: `6379`
- Database: `0`

### 2. **State Updates:**
- El estado se actualiza automáticamente cuando los Agent Nodes modifican `$flow.state`
- Los cambios se persisten automáticamente en Redis
- TTL se renueva en cada interacción

### 3. **Session Management:**
- Cada `sessionId` único mantiene su propio estado
- Para WhatsApp, usa el número de teléfono como sessionId
- Estados expiran después de 24h de inactividad

---

## 🎛️ Configuración Específica por UI

### Si ves estas opciones en Flowise:

**"Enable State"** → ✅ **ACTIVAR**
**"State Type"** → **Persistent** 
**"Storage"** → **Redis** (desde Memory config)
**"Initial State"** → **JSON** (pegar el JSON de arriba)
**"Auto Persist"** → ✅ **ACTIVAR**

---

Esta configuración te dará **memoria persistente completa** para tu agente multiturno de Casalimpia! 🚀