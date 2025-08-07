# Guía completa: Implementación del chatbot multiturno Casalimpia en Flowise v3.0.4

## Tabla de contenidos
1. [Preparación del entorno](#preparación-del-entorno)
2. [Configuración inicial de Flowise](#configuración-inicial-de-flowise)
3. [Configuración de memoria persistente](#configuración-de-memoria-persistente)
4. [Creación del Sequential Agent multiturno](#creación-del-sequential-agent-multiturno)
5. [Implementación de los flujos de conversación](#implementación-de-los-flujos-de-conversación)
6. [Configuración de herramientas personalizadas](#configuración-de-herramientas-personalizadas)
7. [Integración con WhatsApp](#integración-con-whatsapp)
8. [Testing y optimización](#testing-y-optimización)

---

## 1. Preparación del entorno

### 1.1 Instalación de Flowise v3.0.4

**Opción A: Instalación local con NPM**
```bash
# Verificar Node.js (v18.15.0+ o v20+)
node -v
npm -v

# Instalar Flowise globalmente
npm install -g flowise@3.0.4

# Iniciar Flowise
npx flowise start
```

**Opción B: Docker Compose (Recomendado para producción)**
```bash
# Clonar el repositorio
git clone https://github.com/FlowiseAI/Flowise.git
cd Flowise

# Crear archivo .env
cp .env.example .env
```

### 1.2 Configuración del archivo .env

```env
# Puerto del servidor
PORT=3000

# Base de datos (PostgreSQL recomendado para producción)
DATABASE_TYPE=postgres
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=casalimpia_flowise
DATABASE_USER=flowise_user
DATABASE_PASSWORD=your_secure_password

# Rutas de almacenamiento
APIKEY_PATH=/root/.flowise
SECRETKEY_PATH=/root/.flowise
LOG_PATH=/root/.flowise/logs
BLOB_STORAGE_PATH=/root/.flowise/storage

# Autenticación (opcional pero recomendado)
FLOWISE_USERNAME=admin
FLOWISE_PASSWORD=admin123

# Configuración para producción
NODE_ENV=production
```

### 1.3 Configuración de Redis para memoria

**Docker Compose para Redis**
```yaml
# docker-compose.redis.yml
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    
volumes:
  redis_data:
```

**Iniciar Redis**
```bash
docker-compose -f docker-compose.redis.yml up -d
```

---

## 2. Configuración inicial de Flowise

### 2.1 Acceso a la interfaz

1. Abrir navegador en `http://localhost:3000`
2. Hacer login con las credenciales configuradas
3. Ir a **AgentFlow** (no Chatflow para aprovechar Sequential Agents)

### 2.2 Configuración de credenciales

En **Settings → Credentials**, crear:

**OpenAI API Key** (o Claude)
- Nombre: `openai_api_key`
- API Key: `sk-your-openai-key`

**Redis Connection** (para memoria)
- Nombre: `redis_memory`
- Host: `localhost`
- Port: `6379`
- Database: `0`

---

## 3. Configuración de memoria persistente

### 3.1 Configurar Document Store para RAG

1. Ir a **Document Stores**
2. Crear nuevo Document Store:
   - Nombre: `casalimpia_knowledge`
   - Tipo: `Pinecone` o `Chroma` (local)
   - Embeddings: `OpenAI Embeddings`

3. Subir documentos:
   - Información de servicios
   - Ciudades disponibles
   - Precios y políticas
   - FAQs

### 3.2 Configurar Redis Memory

Crear archivo de configuración Redis:
```javascript
// redis-config.js
const redisConfig = {
  host: 'localhost',
  port: 6379,
  db: 0,
  keyPrefix: 'casalimpia:chat:',
  ttl: 86400 // 24 horas
};
```

---

## 4. Creación del Sequential Agent multiturno

### 4.1 Crear nuevo AgentFlow

1. Crear nuevo **AgentFlow**
2. Nombrar: `Casalimpia_Multiturno_Agent`
3. Seleccionar **Sequential Agent V1** architecture

### 4.2 Estructura básica del flujo

#### Start Node (Nodo inicial)
```yaml
Configuración:
- Chat Model: OpenAI GPT-4 o Claude
- System Message: [Tu prompt principal de Casalimpia - ver sección 4.3]
- Memory: Redis Backed Chat Memory
```

#### State Node (Gestión de estado)
```javascript
// Initial State Configuration
{
  "conversation_flow": "initial",
  "user_city": "",
  "selected_service": "",
  "service_date": "",
  "selected_expert": "",
  "cart_items": [],
  "user_cedula": "",
  "user_email": "",
  "current_step": "greeting"
}
```

### 4.3 Prompt principal optimizado

```markdown
# SISTEMA CASALIMPIA - ASISTENTE VIRTUAL ESPECIALIZADO

Eres el asistente virtual especializado en servicios de limpieza para **Casalimpia** (www.casalimpia.com).

## CONTEXTO ACTUAL
- Fecha: {{current_date}}
- Hora: {{current_time}}
- ID Conversación: {{conversation_id}}
- Estado actual: {{$flow.state.current_step}}

## GESTIÓN DE FLUJOS PRINCIPALES

### FLUJO SAC (PRIORIDAD MÁXIMA)
Si detectas: quejas, reclamos, problemas con expertas, reembolsos, incidentes:
**RESPUESTA INMEDIATA**: "Lo siento, no estoy capacitado para responder tu solicitud. Te invito a escribir directamente a nuestra línea de soporte: https://wa.me/573012926214"
**ACCIÓN**: Marcar conversación como "bot_sac" y finalizar.

### FLUJO A: NUEVOS SERVICIOS
**Trigger**: "quiero contratar", "necesito servicio", "comprar limpieza"
**Pasos secuenciales**:
1. Validar/confirmar ciudad (si no está en estado)
2. Mostrar servicios disponibles (usar herramienta get_services)
3. Solicitar fecha de servicio
4. Mostrar expertas disponibles (usar herramienta get_experts)
5. Confirmar y agregar al carrito
6. Ofrecer más servicios o proceder al pago

### FLUJO B: CONSULTA PRECIOS
**Trigger**: "cuánto cuesta", "precios", "tarifas"
**Acción**: Usar herramienta get_services con include_prices=True

### FLUJO C: REAGENDAMIENTO
**Trigger**: "reagendar", "cambiar fecha", "cambiar experta"
**Pasos**:
1. Solicitar cédula
2. Mostrar servicios existentes
3. Confirmar cambios deseados
4. Ejecutar reagendamiento

## HERRAMIENTAS DISPONIBLES
- read_service_cart: Leer estado del carrito
- get_services: Obtener servicios por ciudad
- get_experts: Obtener expertas disponibles
- create_service_cart_item: Agregar servicio al carrito
- reschedule_service: Reagendar servicios

## REGLAS DE COMPORTAMIENTO
- **SIEMPRE** ejecutar read_service_cart al inicio
- **NUNCA** asumir fechas - solicitar confirmación explícita
- Validar ciudades antes de llamar herramientas
- Mantener contexto conversacional
- Responder en español con emojis apropiados
- Ser empático y profesional

## ESTADO ACTUAL
Ciudad establecida: {{$flow.state.user_city}}
Paso actual: {{$flow.state.current_step}}
Servicios en carrito: {{$flow.state.cart_items.length}}
```

---

## 5. Implementación de los flujos de conversación

### 5.1 Condition Agent (Detección de intención)

```javascript
// Condition Agent Configuration
const intentDetection = {
  conditions: [
    {
      condition: "input.includes('quejo') || input.includes('reclamo') || input.includes('problema')",
      action: "sac_flow",
      priority: 1
    },
    {
      condition: "input.includes('precio') || input.includes('costo') || input.includes('tarifa')",
      action: "price_flow",
      priority: 2
    },
    {
      condition: "input.includes('reagendar') || input.includes('cambiar')",
      action: "reschedule_flow",
      priority: 3
    },
    {
      condition: "input.includes('contratar') || input.includes('servicio') || input.includes('comprar')",
      action: "service_flow",
      priority: 4
    }
  ]
};
```

### 5.2 Flujo de servicios nuevos (Sequential)

#### Agent Node 1: Validación de ciudad
```yaml
Configuración:
- Role: "Validador de ciudad"
- Instruction: |
  Extrae y valida la ciudad mencionada por el usuario.
  Ciudades válidas: BOGOTA, MEDELLIN, CALI, BARRANQUILLA, BUCARAMANGA, BOYACA, PEREIRA, VILLAVICENCIO, IBAGUE, CARTAGENA, CHIA, COTA, CAJICA, PALMIRA, JAMUNDI, SANTA MARTA
  
  Si la ciudad es válida, actualiza el estado y continúa.
  Si no es válida, solicita una ciudad válida.
- Tools: [read_service_cart, get_services]
- Update State: |
  {
    "user_city": "{{extracted_city}}",
    "current_step": "city_confirmed"
  }
```

#### Agent Node 2: Selección de servicio
```yaml
Configuración:
- Role: "Consultor de servicios"
- Instruction: |
  Muestra los servicios disponibles para {{$flow.state.user_city}}.
  Presenta en formato numerado y espera selección del usuario.
  NO mostrar campo 'sku' en la respuesta.
- Tools: [get_services]
- Update State: |
  {
    "selected_service": "{{service_selection}}",
    "current_step": "service_selected"
  }
```

#### Agent Node 3: Solicitud de fecha
```yaml
Configuración:
- Role: "Planificador de fechas"
- Instruction: |
  Solicita la fecha deseada para el servicio.
  NUNCA asumas la fecha actual.
  Valida que la fecha sea futura.
  Confirma la fecha en formato completo antes de continuar.
- Update State: |
  {
    "service_date": "{{confirmed_date}}",
    "current_step": "date_confirmed"
  }
```

#### Loop Node: Gestión de múltiples servicios
```yaml
Configuración:
- Condition: "user wants to add more services"
- Max Iterations: 5
- Loop Back To: "Agent Node 2"
```

### 5.3 Condition Nodes para ramificación

#### Condition Node: Verificar carrito
```javascript
// Condition: Check if cart is empty
if (state.cart_items.length === 0) {
  return "empty_cart_flow";
} else {
  return "existing_cart_flow";
}
```

#### Condition Node: Validar ciudad
```javascript
// Condition: Validate city
const validCities = ['BOGOTA', 'MEDELLIN', 'CALI', 'BARRANQUILLA', 'BUCARAMANGA', 'BOYACA', 'PEREIRA', 'VILLAVICENCIO', 'IBAGUE', 'CARTAGENA', 'CHIA', 'COTA', 'CAJICA', 'PALMIRA', 'JAMUNDI', 'SANTA_MARTA'];

if (validCities.includes(state.user_city.toUpperCase())) {
  return "valid_city";
} else {
  return "invalid_city";
}
```

---

## 6. Configuración de herramientas personalizadas

### 6.1 Crear Custom Tools

En **Tools → Custom Tools**, crear las siguientes herramientas:

#### Tool 1: read_service_cart
```javascript
// Custom Tool: read_service_cart
const readServiceCart = async (conversationId) => {
  try {
    const response = await fetch(`${API_BASE}/cart/${conversationId}`, {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${API_TOKEN}`,
        'Content-Type': 'application/json'
      }
    });
    
    const cartData = await response.json();
    return JSON.stringify(cartData);
  } catch (error) {
    return JSON.stringify({ error: 'Failed to read cart', cart_items: [] });
  }
};

// Tool Configuration
{
  "name": "read_service_cart",
  "description": "Lee el estado actual del carrito de servicios del usuario",
  "parameters": {
    "type": "object",
    "properties": {
      "conversation_id": {
        "type": "string",
        "description": "ID de la conversación actual"
      }
    },
    "required": ["conversation_id"]
  }
}
```

#### Tool 2: get_services
```javascript
// Custom Tool: get_services
const getServices = async (city, includePrices = false) => {
  try {
    const response = await fetch(`${API_BASE}/services`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${API_TOKEN}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        ciudad: city.toUpperCase(),
        include_prices: includePrices
      })
    });
    
    const services = await response.json();
    
    // Filtrar el campo 'sku' de la respuesta
    const filteredServices = services.map(service => {
      const { sku, ...serviceWithoutSku } = service;
      return serviceWithoutSku;
    });
    
    return JSON.stringify(filteredServices);
  } catch (error) {
    return JSON.stringify({ error: 'Failed to get services' });
  }
};

// Tool Configuration
{
  "name": "get_services",
  "description": "Obtiene los servicios disponibles para una ciudad específica",
  "parameters": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string",
        "description": "Ciudad para consultar servicios"
      },
      "include_prices": {
        "type": "boolean",
        "description": "Incluir precios en la respuesta",
        "default": false
      }
    },
    "required": ["city"]
  }
}
```

#### Tool 3: get_experts
```javascript
// Custom Tool: get_experts
const getExperts = async (city, date, serviceType) => {
  try {
    const response = await fetch(`${API_BASE}/experts`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${API_TOKEN}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        ciudad: city,
        fecha: date,
        tipo_servicio: serviceType
      })
    });
    
    const experts = await response.json();
    
    // Filtrar el campo 'cedula' de la respuesta
    const filteredExperts = experts.map(expert => {
      const { cedula, ...expertWithoutCedula } = expert;
      return expertWithoutCedula;
    });
    
    return JSON.stringify(filteredExperts);
  } catch (error) {
    return JSON.stringify({ error: 'Failed to get experts' });
  }
};
```

### 6.2 Variables de entorno para APIs

```env
# API Configuration
CASALIMPIA_API_BASE=https://api.casalimpia.com/v1
CASALIMPIA_API_TOKEN=your_api_token_here
```

---

## 7. Integración con WhatsApp

### 7.1 Opción A: Integración vía Make.com (Recomendado)

#### Paso 1: Configurar Meta App
1. Ir a [developers.facebook.com](https://developers.facebook.com)
2. Crear nueva app → Business → WhatsApp
3. Configurar WhatsApp Product
4. Obtener **Access Token** y **Phone Number ID**

#### Paso 2: Configurar webhook en Make.com
```javascript
// Make.com Scenario Configuration
{
  "trigger": {
    "type": "webhook",
    "url": "https://hook.make.com/your-webhook-url"
  },
  "modules": [
    {
      "type": "webhook_receive",
      "name": "WhatsApp Incoming Message"
    },
    {
      "type": "http_request",
      "name": "Call Flowise",
      "url": "https://your-flowise-instance.com/api/v1/prediction/your-flow-id",
      "method": "POST",
      "headers": {
        "Content-Type": "application/json"
      },
      "body": {
        "question": "{{message.text}}",
        "overrideConfig": {
          "sessionId": "{{message.from}}"
        }
      }
    },
    {
      "type": "http_request",
      "name": "Send WhatsApp Reply",
      "url": "https://graph.facebook.com/v19.0/{{phone_number_id}}/messages",
      "headers": {
        "Authorization": "Bearer {{access_token}}"
      },
      "body": {
        "messaging_product": "whatsapp",
        "to": "{{message.from}}",
        "text": {
          "body": "{{flowise_response.text}}"
        }
      }
    }
  ]
}
```

### 7.2 Opción B: Integración vía Zapier

#### Configuración del Zap:
1. **Trigger**: WhatsApp Business → New Message Received
2. **Action 1**: HTTP Request → Call Flowise API
3. **Action 2**: WhatsApp Business → Send Message

```javascript
// Zapier HTTP Request Configuration
{
  "url": "https://your-flowise-instance.com/api/v1/prediction/your-flow-id",
  "method": "POST",
  "headers": {
    "Content-Type": "application/json"
  },
  "data": {
    "question": "{{trigger.message}}",
    "overrideConfig": {
      "sessionId": "{{trigger.phone_number}}"
    }
  }
}
```

### 7.3 Configuración de Session ID

En el AgentFlow, configurar el **Session ID** para memoria:

```yaml
Redis Memory Configuration:
- Session ID: "{{overrideConfig.sessionId}}"
- Key Prefix: "casalimpia:whatsapp:"
- TTL: 86400 (24 horas)
```

---

## 8. Testing y optimización

### 8.1 Testing de flujos individuales

#### Test 1: Flujo básico de compra
```yaml
Test Case: "Quiero contratar un servicio en Bogotá"
Expected Flow:
1. Confirmar ciudad → ✅
2. Mostrar servicios → ✅
3. Seleccionar servicio → ✅
4. Solicitar fecha → ✅
5. Mostrar expertas → ✅
6. Confirmar carrito → ✅
```

#### Test 2: Gestión de estado
```yaml
Test Case: Conversación multiturno con interrupciones
Scenario:
1. Usuario: "Hola"
2. Bot: Menú de opciones
3. Usuario: "Precios en Cali"
4. Bot: Lista de precios
5. Usuario: "Quiero el de 4 horas"
Expected: Bot recuerda el contexto de Cali
```

#### Test 3: Validaciones
```yaml
Test Cases:
- Ciudad inválida: "Servicio en París" → Error esperado
- Fecha pasada: "Para ayer" → Error esperado  
- Cédula inválida: "123" → Error esperado
```

### 8.2 Configuración de logs y monitoreo

#### Logging configuration
```javascript
// En el .env file
LOG_LEVEL=debug
LOG_PATH=/root/.flowise/logs

// Custom logging en tools
const logger = {
  info: (message, data) => {
    console.log(`[INFO] ${new Date().toISOString()} - ${message}`, data);
  },
  error: (message, error) => {
    console.error(`[ERROR] ${new Date().toISOString()} - ${message}`, error);
  }
};
```

### 8.3 Optimización de rendimiento

#### Memory optimization
```yaml
Redis Configuration:
- Max Memory: 256mb
- Memory Policy: allkeys-lru
- Save: 900 1 300 10 60 10000

PostgreSQL Configuration:
- shared_buffers: 256MB
- effective_cache_size: 1GB
- work_mem: 4MB
```

#### Prompt optimization
```markdown
# Técnicas de optimización:
1. **Chunking semántico**: Dividir prompt en secciones lógicas
2. **Caching de respuestas**: Para consultas frecuentes
3. **Lazy loading**: Cargar contexto solo cuando es necesario
4. **Compresión de estado**: Usar códigos para estados frecuentes
```

---

## 9. Deployment y producción

### 9.1 Docker Compose para producción

```yaml
# docker-compose.prod.yml
version: '3.8'
services:
  flowise:
    image: flowiseai/flowise:3.0.4
    restart: always
    environment:
      - PORT=3000
      - DATABASE_TYPE=postgres
      - DATABASE_HOST=postgres
      - DATABASE_NAME=casalimpia_flowise
      - DATABASE_USER=flowise
      - DATABASE_PASSWORD=${DB_PASSWORD}
      - REDIS_URL=redis://redis:6379
    ports:
      - "3000:3000"
    volumes:
      - flowise_data:/root/.flowise
    depends_on:
      - postgres
      - redis
    
  postgres:
    image: postgres:15
    restart: always
    environment:
      - POSTGRES_DB=casalimpia_flowise
      - POSTGRES_USER=flowise
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      
  redis:
    image: redis:7-alpine
    restart: always
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    
  nginx:
    image: nginx:alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/ssl
      
volumes:
  flowise_data:
  postgres_data:
  redis_data:
```

### 9.2 Nginx configuration

```nginx
# nginx.conf
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name your-domain.com;
    
    ssl_certificate /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;
    
    location / {
        proxy_pass http://flowise:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 9.3 Backup y recovery

```bash
#!/bin/bash
# backup.sh

# Backup PostgreSQL
pg_dump -h localhost -U flowise casalimpia_flowise > backup_$(date +%Y%m%d_%H%M%S).sql

# Backup Redis
redis-cli --rdb backup_redis_$(date +%Y%m%d_%H%M%S).rdb

# Backup Flowise data
tar -czf backup_flowise_$(date +%Y%m%d_%H%M%S).tar.gz /root/.flowise
```

---

## 10. Troubleshooting común

### 10.1 Problemas frecuentes

**Error: "Cannot read property of undefined"**
```javascript
// Solución: Validar estado antes de acceder
if (state && state.user_city) {
  // Acceder a state.user_city
}
```

**Error: "Redis connection failed"**
```bash
# Verificar conexión Redis
redis-cli ping
# Debe responder: PONG
```

**Error: "Tool execution timeout"**
```yaml
# Aumentar timeout en tool configuration
Tool Configuration:
  timeout: 30000  # 30 segundos
```

### 10.2 Debugging tips

```javascript
// Agregar logs en custom tools
console.log('State before tool execution:', JSON.stringify(state, null, 2));
console.log('Tool input:', JSON.stringify(input, null, 2));
console.log('Tool output:', JSON.stringify(output, null, 2));
```

### 10.3 Performance monitoring

```javascript
// Métricas a monitorear:
const metrics = {
  response_time: "< 3 segundos",
  memory_usage: "< 512MB",
  redis_memory: "< 256MB",
  conversation_completion_rate: "> 80%",
  error_rate: "< 5%"
};
```

---

Esta guía te proporciona una base sólida para implementar tu chatbot multiturno de Casalimpia en Flowise v3.0.4. La arquitectura de Sequential Agents te permitirá manejar los flujos complejos mientras mantienes flexibilidad para futuras expansiones.

Para siguientes pasos, recomiendo:
1. Implementar primero el flujo básico de compra
2. Agregar gradualmente los otros flujos
3. Realizar testing exhaustivo antes de producción
4. Monitorear métricas continuamente post-deployment