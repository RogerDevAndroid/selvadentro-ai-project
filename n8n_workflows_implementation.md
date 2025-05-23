# Implementación del Squad Colaborativo de Agentes de IA en n8n

## Introducción a n8n para el Squad Colaborativo

[n8n](https://n8n.io/) es una plataforma de automatización de flujos de trabajo que nos permitirá implementar el Squad Colaborativo de Agentes de IA para Selvadentro Tulum de manera efectiva. Con n8n, podemos:

1. Orquestar la comunicación entre los diferentes agentes especializados
2. Integrar APIs externas (WhatsApp Business API, Vapi, CRMs, etc.)
3. Implementar lógica condicional para el enrutamiento de mensajes
4. Almacenar y recuperar datos de la base de conocimiento centralizada
5. Programar tareas recurrentes para el aprendizaje y optimización

Este documento detalla los workflows principales que necesitamos crear en n8n para implementar el Squad Colaborativo completo.

## Arquitectura General de Workflows en n8n

![Arquitectura de Workflows n8n](squad_colaborativo_arquitectura.png)

Implementaremos los siguientes workflows principales en n8n:

1. **Workflow de Orquestación Principal**
2. **Workflow de Procesamiento de Mensajes de WhatsApp**
3. **Workflow de Gestión de Llamadas Vapi**
4. **Workflow de Consulta a la Base de Conocimiento**
5. **Workflow de Perfilado de Clientes**
6. **Workflow de Manejo de Objeciones**
7. **Workflow de Auditoría y Aprendizaje**
8. **Workflow de Handover al Equipo Humano**

## 1. Workflow de Orquestación Principal

Este workflow actúa como el "Agente Orquestador" del squad, coordinando todas las interacciones y el flujo de información.

### Nodos del Workflow:

```
[Webhook Trigger] → [Switch: Tipo de Entrada] → [Función: Analizar Contenido] → [Switch: Acción Requerida] → [Múltiples Salidas a Otros Workflows]
```

### Configuración Detallada:

#### Webhook Trigger
- **Método**: POST
- **Ruta**: `/orchestrator`
- **Autenticación**: Bearer Token

#### Switch: Tipo de Entrada
- **Opciones**:
  - Mensaje de WhatsApp
  - Llamada de Voz
  - Evento de CRM
  - Solicitud Interna

#### Función: Analizar Contenido
```javascript
// Código para analizar el contenido del mensaje/evento
const input = items[0].json;
const content = input.content;
const metadata = input.metadata;

// Determinar intención y urgencia
let intent = "unknown";
let priority = "normal";

// Análisis básico de intención mediante palabras clave
if (content.toLowerCase().includes("precio") || content.toLowerCase().includes("costo")) {
  intent = "pricing_inquiry";
}
else if (content.toLowerCase().includes("visita") || content.toLowerCase().includes("cita")) {
  intent = "appointment_request";
  priority = "high";
}
else if (content.toLowerCase().includes("seguridad") || content.toLowerCase().includes("legal")) {
  intent = "objection_handling";
}
// ... más reglas de análisis

return {
  json: {
    ...input,
    analyzed: {
      intent: intent,
      priority: priority,
      timestamp: new Date().toISOString()
    }
  }
};
```

#### Switch: Acción Requerida
- **Opciones basadas en la intención detectada**:
  - Respuesta a Consulta General → Workflow de WhatsApp
  - Cualificación de Lead → Workflow de Perfilado
  - Manejo de Objeción → Workflow de Objeciones
  - Agendamiento → Workflow de Gestión de Citas
  - Handover a Humano → Workflow de Handover

## 2. Workflow de Procesamiento de Mensajes de WhatsApp

Este workflow implementa el "Agente WhatsApp" que maneja todas las interacciones escritas.

### Nodos del Workflow:

```
[Webhook Trigger] → [HTTP Request: WhatsApp Business API] → [Función: Extraer Mensaje] → [HTTP Request: OpenAI API] → [Función: Formatear Respuesta] → [HTTP Request: Enviar Respuesta WhatsApp] → [Función: Registrar en Base de Conocimiento]
```

### Configuración Detallada:

#### Webhook Trigger
- **Método**: POST
- **Ruta**: `/whatsapp/incoming`
- **Autenticación**: Configurada según WhatsApp Business API

#### HTTP Request: WhatsApp Business API
- **Método**: GET
- **URL**: `https://graph.facebook.com/v17.0/{{$node["Webhook Trigger"].json["entry"][0]["changes"][0]["value"]["metadata"]["phone_number_id"]}}/messages`
- **Headers**: Autenticación con token de WhatsApp Business API

#### Función: Extraer Mensaje
```javascript
// Extraer el mensaje y metadatos relevantes
const input = items[0].json;
const messageData = input.entry[0].changes[0].value;
const message = messageData.messages[0];
const sender = message.from;
const messageText = message.text?.body || "";

return {
  json: {
    sender: sender,
    message: messageText,
    timestamp: message.timestamp,
    messageId: message.id,
    metadata: {
      source: "whatsapp",
      waId: messageData.metadata.phone_number_id
    }
  }
};
```

#### HTTP Request: OpenAI API
- **Método**: POST
- **URL**: `https://api.openai.com/v1/chat/completions`
- **Headers**: 
  - Authorization: Bearer {{$env.OPENAI_API_KEY}}
  - Content-Type: application/json
- **Body**:
```json
{
  "model": "gpt-4",
  "messages": [
    {
      "role": "system",
      "content": "Eres SelvaConcierge, el asistente virtual exclusivo para Selvadentro Tulum, un prestigioso desarrollo inmobiliario de lujo que ofrece lotes privados en la selva con cenotes naturales en Tulum, México. Tu comunicación debe ser sofisticada, serena y empática, evocando la experiencia de santuario exclusivo que define Selvadentro. Sé impecablemente cortés y demuestra un interés auténtico en la visión del cliente."
    },
    {
      "role": "user",
      "content": "{{$node["Función: Extraer Mensaje"].json["message"]}}"
    }
  ],
  "temperature": 0.7,
  "max_tokens": 500
}
```

#### Función: Formatear Respuesta
```javascript
// Formatear la respuesta para WhatsApp
const input = items[0].json;
const aiResponse = input.choices[0].message.content;

// Limitar longitud para WhatsApp si es necesario
const formattedResponse = aiResponse.length > 1000 ? 
  aiResponse.substring(0, 997) + "..." : 
  aiResponse;

return {
  json: {
    recipient: $node["Función: Extraer Mensaje"].json.sender,
    message: formattedResponse,
    metadata: $node["Función: Extraer Mensaje"].json.metadata
  }
};
```

#### HTTP Request: Enviar Respuesta WhatsApp
- **Método**: POST
- **URL**: `https://graph.facebook.com/v17.0/{{$node["Función: Extraer Mensaje"].json.metadata.waId}}/messages`
- **Headers**: Autenticación con token de WhatsApp Business API
- **Body**:
```json
{
  "messaging_product": "whatsapp",
  "recipient_type": "individual",
  "to": "{{$node["Función: Formatear Respuesta"].json.recipient}}",
  "type": "text",
  "text": {
    "body": "{{$node["Función: Formatear Respuesta"].json.message}}"
  }
}
```

#### Función: Registrar en Base de Conocimiento
```javascript
// Registrar la interacción en la base de conocimiento
const interaction = {
  type: "whatsapp",
  client: $node["Función: Extraer Mensaje"].json.sender,
  clientMessage: $node["Función: Extraer Mensaje"].json.message,
  agentResponse: $node["Función: Formatear Respuesta"].json.message,
  timestamp: new Date().toISOString(),
  metadata: $node["Función: Extraer Mensaje"].json.metadata
};

// Aquí iría el código para guardar en la base de datos vectorial
// Por ejemplo, usando Pinecone, Weaviate, etc.

return {
  json: {
    status: "recorded",
    interactionId: generateUUID(), // Función para generar ID único
    interaction: interaction
  }
};
```

## 3. Workflow de Gestión de Llamadas Vapi

Este workflow implementa el "Agente Vapi" que maneja las interacciones de voz.

### Nodos del Workflow:

```
[Webhook Trigger] → [HTTP Request: Vapi API] → [Función: Configurar Llamada] → [HTTP Request: Iniciar Llamada] → [Webhook: Recibir Eventos de Llamada] → [Switch: Tipo de Evento] → [Función: Procesar Respuesta] → [HTTP Request: Actualizar Llamada] → [Función: Registrar en Base de Conocimiento]
```

### Configuración Detallada:

#### Webhook Trigger
- **Método**: POST
- **Ruta**: `/voice/initiate`

#### HTTP Request: Vapi API
- **Método**: POST
- **URL**: `https://api.vapi.ai/call/start`
- **Headers**: 
  - Authorization: Bearer {{$env.VAPI_API_KEY}}
  - Content-Type: application/json
- **Body**:
```json
{
  "phoneNumber": "{{$node["Webhook Trigger"].json.phoneNumber}}",
  "callerId": "{{$env.SELVADENTRO_CALLER_ID}}",
  "promptId": "{{$env.SELVADENTRO_PROMPT_ID}}",
  "firstMessage": "Hola, soy el asistente virtual de Selvadentro Tulum. Gracias por su interés en nuestro exclusivo desarrollo. ¿En qué puedo ayudarle hoy?",
  "webhookUrl": "{{$env.WEBHOOK_BASE_URL}}/voice/events"
}
```

#### Webhook: Recibir Eventos de Llamada
- **Método**: POST
- **Ruta**: `/voice/events`

#### Switch: Tipo de Evento
- **Opciones**:
  - call.started
  - call.ended
  - transcription
  - human.needed

#### Función: Procesar Respuesta (para evento "transcription")
```javascript
// Procesar la transcripción y generar respuesta
const input = items[0].json;
const transcription = input.transcription;
const callId = input.callId;

// Aquí iría la lógica para generar la respuesta basada en la transcripción
// Podría incluir llamadas a OpenAI, consultas a la base de conocimiento, etc.

return {
  json: {
    callId: callId,
    response: "Gracias por su interés en el lote B-12. Este lote tiene una superficie de 1,673 m² y está ubicado junto al cenote principal. ¿Le gustaría agendar una visita virtual para conocerlo en detalle?",
    action: "continue"
  }
};
```

#### HTTP Request: Actualizar Llamada
- **Método**: POST
- **URL**: `https://api.vapi.ai/call/{{$node["Función: Procesar Respuesta"].json.callId}}/respond`
- **Headers**: 
  - Authorization: Bearer {{$env.VAPI_API_KEY}}
  - Content-Type: application/json
- **Body**:
```json
{
  "response": "{{$node["Función: Procesar Respuesta"].json.response}}",
  "action": "{{$node["Función: Procesar Respuesta"].json.action}}"
}
```

## 4. Workflow de Consulta a la Base de Conocimiento

Este workflow implementa el "Agente de Conocimiento" que mantiene y consulta la información actualizada del proyecto.

### Nodos del Workflow:

```
[Webhook Trigger] → [Función: Preparar Consulta] → [HTTP Request: Vector Database] → [Función: Procesar Resultados] → [Respond to Webhook]
```

### Configuración Detallada:

#### Webhook Trigger
- **Método**: POST
- **Ruta**: `/knowledge/query`

#### Función: Preparar Consulta
```javascript
// Preparar la consulta para la base de datos vectorial
const input = items[0].json;
const query = input.query;
const category = input.category || "general";
const limit = input.limit || 5;

// Generar embedding para la consulta (simplificado)
// En la implementación real, esto podría ser una llamada a OpenAI Embeddings API
const embedding = generateEmbedding(query);

return {
  json: {
    vector: embedding,
    namespace: category,
    topK: limit,
    includeMetadata: true,
    originalQuery: query
  }
};
```

#### HTTP Request: Vector Database (ejemplo con Pinecone)
- **Método**: POST
- **URL**: `https://{{$env.PINECONE_INDEX}}-{{$env.PINECONE_PROJECT}}.svc.{{$env.PINECONE_ENVIRONMENT}}.pinecone.io/query`
- **Headers**: 
  - Api-Key: {{$env.PINECONE_API_KEY}}
  - Content-Type: application/json
- **Body**:
```json
{
  "vector": {{$node["Función: Preparar Consulta"].json.vector}},
  "namespace": "{{$node["Función: Preparar Consulta"].json.namespace}}",
  "topK": {{$node["Función: Preparar Consulta"].json.topK}},
  "includeMetadata": true
}
```

#### Función: Procesar Resultados
```javascript
// Procesar los resultados de la consulta
const input = items[0].json;
const matches = input.matches || [];
const originalQuery = $node["Función: Preparar Consulta"].json.originalQuery;

// Extraer y formatear la información relevante
const results = matches.map(match => {
  return {
    content: match.metadata.content,
    source: match.metadata.source,
    relevanceScore: match.score,
    id: match.id
  };
});

return {
  json: {
    query: originalQuery,
    results: results,
    timestamp: new Date().toISOString()
  }
};
```

## 5. Workflow de Perfilado de Clientes

Este workflow implementa el "Agente de Perfilado" que identifica y clasifica buyer personas.

### Nodos del Workflow:

```
[Webhook Trigger] → [HTTP Request: Obtener Historial] → [HTTP Request: OpenAI API] → [Función: Extraer Perfil] → [HTTP Request: Actualizar CRM] → [Respond to Webhook]
```

### Configuración Detallada:

#### Webhook Trigger
- **Método**: POST
- **Ruta**: `/profiling/analyze`

#### HTTP Request: Obtener Historial
- **Método**: GET
- **URL**: `{{$env.API_BASE_URL}}/clients/{{$node["Webhook Trigger"].json.clientId}}/interactions`
- **Headers**: Autenticación apropiada

#### HTTP Request: OpenAI API
- **Método**: POST
- **URL**: `https://api.openai.com/v1/chat/completions`
- **Headers**: 
  - Authorization: Bearer {{$env.OPENAI_API_KEY}}
  - Content-Type: application/json
- **Body**:
```json
{
  "model": "gpt-4",
  "messages": [
    {
      "role": "system",
      "content": "Eres un experto en análisis de perfiles de clientes para el sector inmobiliario de lujo. Tu tarea es analizar las interacciones de un cliente potencial y determinar a qué buyer persona se asemeja más, su nivel de conciencia actual, y sus principales motivaciones e intereses. Utiliza los siguientes perfiles de buyer personas como referencia:\n\n1. Elena, la Eco-Lujosa Consciente: Europea, 45-55 años, emprendedora en tecnología sostenible, valora sostenibilidad, privacidad y bienestar holístico.\n\n2. David, el Inversor en Bienestar Familiar: Norteamericano, 40-50 años, ejecutivo de alto nivel, busca inversión segura y espacio para reconexión familiar.\n\n3. Carlos, el Coleccionista de Experiencias Exclusivas: Latinoamericano, 35-45 años, empresario exitoso, busca propiedades únicas y experiencias auténticas.\n\nNiveles de conciencia:\n1. Inconsciente del Problema/Necesidad\n2. Consciente del Problema\n3. Consciente de la Solución\n4. Consciente del Producto\n5. Totalmente Consciente/Listo para Comprar"
    },
    {
      "role": "user",
      "content": "Analiza las siguientes interacciones del cliente y determina su perfil, nivel de conciencia, motivaciones principales y posibles objeciones:\n\n{{$node["HTTP Request: Obtener Historial"].json | stringify}}"
    }
  ],
  "temperature": 0.3,
  "max_tokens": 1000
}
```

#### Función: Extraer Perfil
```javascript
// Extraer el perfil del cliente de la respuesta de OpenAI
const input = items[0].json;
const analysis = input.choices[0].message.content;

// Parsear el análisis para extraer información estructurada
// Este es un ejemplo simplificado; en la implementación real
// se utilizaría un parsing más robusto o se solicitaría a OpenAI
// que devuelva directamente un formato estructurado

const buyerPersonaMatch = analysis.match(/Buyer Persona: (.*?)(?:\n|$)/);
const awarenessLevelMatch = analysis.match(/Nivel de Conciencia: (.*?)(?:\n|$)/);
const motivationsMatch = analysis.match(/Motivaciones Principales: (.*?)(?:\n|$)/);
const objectionsMatch = analysis.match(/Posibles Objeciones: (.*?)(?:\n|$)/);

return {
  json: {
    clientId: $node["Webhook Trigger"].json.clientId,
    profile: {
      buyerPersona: buyerPersonaMatch ? buyerPersonaMatch[1].trim() : "No determinado",
      awarenessLevel: awarenessLevelMatch ? awarenessLevelMatch[1].trim() : "No determinado",
      motivations: motivationsMatch ? motivationsMatch[1].trim() : "No determinadas",
      potentialObjections: objectionsMatch ? objectionsMatch[1].trim() : "No determinadas"
    },
    rawAnalysis: analysis,
    timestamp: new Date().toISOString()
  }
};
```

#### HTTP Request: Actualizar CRM
- **Método**: PUT
- **URL**: `{{$env.API_BASE_URL}}/clients/{{$node["Función: Extraer Perfil"].json.clientId}}/profile`
- **Headers**: Autenticación apropiada
- **Body**:
```json
{
  "buyerPersona": "{{$node["Función: Extraer Perfil"].json.profile.buyerPersona}}",
  "awarenessLevel": "{{$node["Función: Extraer Perfil"].json.profile.awarenessLevel}}",
  "motivations": "{{$node["Función: Extraer Perfil"].json.profile.motivations}}",
  "potentialObjections": "{{$node["Función: Extraer Perfil"].json.profile.potentialObjections}}",
  "lastUpdated": "{{$node["Función: Extraer Perfil"].json.timestamp}}"
}
```

## 6. Workflow de Manejo de Objeciones

Este workflow implementa el "Agente de Objeciones" que maneja objeciones específicas del sector inmobiliario de lujo.

### Nodos del Workflow:

```
[Webhook Trigger] → [Función: Clasificar Objeción] → [HTTP Request: Consultar Base de Conocimiento] → [HTTP Request: OpenAI API] → [Función: Formatear Respuesta] → [Respond to Webhook]
```

### Configuración Detallada:

#### Webhook Trigger
- **Método**: POST
- **Ruta**: `/objections/handle`

#### Función: Clasificar Objeción
```javascript
// Clasificar el tipo de objeción
const input = items[0].json;
const objectionText = input.objectionText;
const clientProfile = input.clientProfile || {};

// Categorías comunes de objeciones
const categories = {
  "seguridad_inversion": ["seguridad", "riesgo", "estafa", "fraude", "legal", "fideicomiso"],
  "precio": ["caro", "precio", "costoso", "presupuesto", "descuento", "financiamiento"],
  "ubicacion": ["lejos", "distancia", "acceso", "infraestructura", "servicios"],
  "impacto_ambiental": ["ambiente", "ecológico", "sostenible", "cenote", "selva", "impacto"],
  "retorno_inversion": ["retorno", "apreciación", "rentabilidad", "roi", "inversión"]
};

// Determinar categoría
let objectionCategory = "general";
for (const [category, keywords] of Object.entries(categories)) {
  if (keywords.some(keyword => objectionText.toLowerCase().includes(keyword))) {
    objectionCategory = category;
    break;
  }
}

return {
  json: {
    ...input,
    objectionCategory: objectionCategory
  }
};
```

#### HTTP Request: Consultar Base de Conocimiento
- **Método**: POST
- **URL**: `{{$env.API_BASE_URL}}/knowledge/query`
- **Headers**: Autenticación apropiada
- **Body**:
```json
{
  "query": "{{$node["Función: Clasificar Objeción"].json.objectionText}}",
  "category": "objections_{{$node["Función: Clasificar Objeción"].json.objectionCategory}}",
  "limit": 3
}
```

#### HTTP Request: OpenAI API
- **Método**: POST
- **URL**: `https://api.openai.com/v1/chat/completions`
- **Headers**: 
  - Authorization: Bearer {{$env.OPENAI_API_KEY}}
  - Content-Type: application/json
- **Body**:
```json
{
  "model": "gpt-4",
  "messages": [
    {
      "role": "system",
      "content": "Eres un experto en ventas inmobiliarias de lujo especializado en manejar objeciones. Tu tarea es crear una respuesta empática, informativa y persuasiva a una objeción planteada por un cliente potencial de Selvadentro Tulum, un desarrollo de lotes selváticos de lujo con cenotes privados.\n\nEstructura tu respuesta en tres partes:\n1. Validación: Reconoce la preocupación y muestra empatía\n2. Información: Proporciona datos relevantes y contexto\n3. Reencuadre: Transforma la objeción en una ventaja o oportunidad\n\nUtiliza un tono sofisticado, sereno y confiado, apropiado para comunicarse con clientes de alto patrimonio."
    },
    {
      "role": "user",
      "content": "Responde a la siguiente objeción de un cliente potencial. Utiliza la información de referencia proporcionada.\n\nObjeción: {{$node["Función: Clasificar Objeción"].json.objectionText}}\n\nCategoría: {{$node["Función: Clasificar Objeción"].json.objectionCategory}}\n\nPerfil del cliente: {{$node["Función: Clasificar Objeción"].json.clientProfile | stringify}}\n\nInformación de referencia: {{$node["HTTP Request: Consultar Base de Conocimiento"].json | stringify}}"
    }
  ],
  "temperature": 0.5,
  "max_tokens": 800
}
```

#### Función: Formatear Respuesta
```javascript
// Formatear la respuesta a la objeción
const input = items[0].json;
const response = input.choices[0].message.content;

return {
  json: {
    objectionText: $node["Función: Clasificar Objeción"].json.objectionText,
    objectionCategory: $node["Función: Clasificar Objeción"].json.objectionCategory,
    response: response,
    timestamp: new Date().toISOString()
  }
};
```

## 7. Workflow de Auditoría y Aprendizaje

Este workflow implementa el "Agente Auditor" que analiza las interacciones y genera recomendaciones para mejorar.

### Nodos del Workflow:

```
[Cron Trigger] → [HTTP Request: Obtener Interacciones Recientes] → [Función: Preparar Análisis] → [HTTP Request: OpenAI API] → [Función: Extraer Recomendaciones] → [HTTP Request: Actualizar Base de Conocimiento] → [HTTP Request: Notificar Equipo]
```

### Configuración Detallada:

#### Cron Trigger
- **Frecuencia**: Diaria a las 01:00 AM

#### HTTP Request: Obtener Interacciones Recientes
- **Método**: GET
- **URL**: `{{$env.API_BASE_URL}}/interactions?since={{$formatDate($now.minus({ days: 1 }), "YYYY-MM-DD")}}`
- **Headers**: Autenticación apropiada

#### Función: Preparar Análisis
```javascript
// Preparar los datos para análisis
const input = items[0].json;
const interactions = input.interactions || [];

// Agrupar interacciones por tipo
const whatsappInteractions = interactions.filter(i => i.type === "whatsapp");
const voiceInteractions = interactions.filter(i => i.type === "voice");
const handoverInteractions = interactions.filter(i => i.handoverTriggered === true);

// Extraer métricas básicas
const metrics = {
  totalInteractions: interactions.length,
  whatsappCount: whatsappInteractions.length,
  voiceCount: voiceInteractions.length,
  handoverCount: handoverInteractions.length,
  successfulQualifications: interactions.filter(i => i.qualificationSuccess === true).length,
  appointmentsScheduled: interactions.filter(i => i.appointmentScheduled === true).length
};

return {
  json: {
    date: $formatDate($now, "YYYY-MM-DD"),
    metrics: metrics,
    interactions: interactions.slice(0, 50) // Limitar a 50 para no exceder tokens
  }
};
```

#### HTTP Request: OpenAI API
- **Método**: POST
- **URL**: `https://api.openai.com/v1/chat/completions`
- **Headers**: 
  - Authorization: Bearer {{$env.OPENAI_API_KEY}}
  - Content-Type: application/json
- **Body**:
```json
{
  "model": "gpt-4",
  "messages": [
    {
      "role": "system",
      "content": "Eres un auditor experto en ventas inmobiliarias de lujo y experiencia del cliente. Tu tarea es analizar las interacciones recientes entre los agentes de IA de Selvadentro Tulum y los clientes potenciales para identificar patrones, fortalezas, debilidades y oportunidades de mejora. Proporciona recomendaciones específicas y accionables para optimizar el rendimiento del sistema."
    },
    {
      "role": "user",
      "content": "Analiza las siguientes interacciones y métricas del día {{$node["Función: Preparar Análisis"].json.date}}. Proporciona un análisis detallado y recomendaciones estructuradas.\n\nMétricas:\n{{$node["Función: Preparar Análisis"].json.metrics | stringify}}\n\nInteracciones:\n{{$node["Función: Preparar Análisis"].json.interactions | stringify}}"
    }
  ],
  "temperature": 0.3,
  "max_tokens": 1500
}
```

#### Función: Extraer Recomendaciones
```javascript
// Extraer recomendaciones del análisis
const input = items[0].json;
const analysis = input.choices[0].message.content;

// Aquí se podría implementar un parsing más sofisticado
// para extraer recomendaciones específicas por categoría

return {
  json: {
    date: $node["Función: Preparar Análisis"].json.date,
    metrics: $node["Función: Preparar Análisis"].json.metrics,
    analysis: analysis,
    timestamp: new Date().toISOString()
  }
};
```

#### HTTP Request: Actualizar Base de Conocimiento
- **Método**: POST
- **URL**: `{{$env.API_BASE_URL}}/knowledge/audit`
- **Headers**: Autenticación apropiada
- **Body**:
```json
{
  "date": "{{$node["Función: Extraer Recomendaciones"].json.date}}",
  "analysis": "{{$node["Función: Extraer Recomendaciones"].json.analysis}}",
  "metrics": {{$node["Función: Extraer Recomendaciones"].json.metrics | stringify}}
}
```

#### HTTP Request: Notificar Equipo
- **Método**: POST
- **URL**: `{{$env.API_BASE_URL}}/notifications/team`
- **Headers**: Autenticación apropiada
- **Body**:
```json
{
  "type": "audit_report",
  "title": "Reporte de Auditoría de Agentes IA - {{$node["Función: Extraer Recomendaciones"].json.date}}",
  "content": "Se ha generado el reporte de auditoría diario. Principales hallazgos:\n\n{{$node["Función: Extraer Recomendaciones"].json.analysis.substring(0, 500)}}...\n\nConsulte el reporte completo en el dashboard.",
  "priority": "normal",
  "recipients": ["sales_manager", "ai_team"]
}
```

## 8. Workflow de Handover al Equipo Humano

Este workflow gestiona la transferencia de interacciones del sistema de IA al equipo humano.

### Nodos del Workflow:

```
[Webhook Trigger] → [Función: Preparar Contexto] → [HTTP Request: Notificar Asesor] → [HTTP Request: Actualizar CRM] → [Función: Generar Respuesta Transitoria] → [Respond to Webhook]
```

### Configuración Detallada:

#### Webhook Trigger
- **Método**: POST
- **Ruta**: `/handover/initiate`

#### Función: Preparar Contexto
```javascript
// Preparar el contexto para el handover
const input = items[0].json;
const clientId = input.clientId;
const channelType = input.channelType; // "whatsapp" o "voice"
const reason = input.reason;
const interactionHistory = input.interactionHistory || [];

// Generar un resumen conciso para el asesor humano
let summary = `Cliente ID: ${clientId}\n`;
summary += `Canal: ${channelType}\n`;
summary += `Motivo de transferencia: ${reason}\n\n`;
summary += "Resumen de la conversación:\n";

// Añadir los últimos 5 intercambios (o menos si hay menos)
const recentExchanges = interactionHistory.slice(-5);
for (const exchange of recentExchanges) {
  summary += `- Cliente: ${exchange.clientMessage}\n`;
  summary += `- Agente IA: ${exchange.agentResponse}\n\n`;
}

// Determinar el asesor más adecuado basado en disponibilidad y especialización
// En una implementación real, esto podría ser una llamada a un servicio de asignación
const assignedAdvisor = determineAdvisor(input);

return {
  json: {
    ...input,
    summary: summary,
    assignedAdvisor: assignedAdvisor,
    handoverId: generateUUID(),
    timestamp: new Date().toISOString()
  }
};
```

#### HTTP Request: Notificar Asesor
- **Método**: POST
- **URL**: `{{$env.API_BASE_URL}}/advisors/{{$node["Función: Preparar Contexto"].json.assignedAdvisor.id}}/notifications`
- **Headers**: Autenticación apropiada
- **Body**:
```json
{
  "type": "handover_request",
  "handoverId": "{{$node["Función: Preparar Contexto"].json.handoverId}}",
  "clientId": "{{$node["Función: Preparar Contexto"].json.clientId}}",
  "channelType": "{{$node["Función: Preparar Contexto"].json.channelType}}",
  "summary": "{{$node["Función: Preparar Contexto"].json.summary}}",
  "priority": "{{$node["Función: Preparar Contexto"].json.reason === 'high_intent' ? 'high' : 'normal'}}",
  "timestamp": "{{$node["Función: Preparar Contexto"].json.timestamp}}"
}
```

#### HTTP Request: Actualizar CRM
- **Método**: POST
- **URL**: `{{$env.API_BASE_URL}}/clients/{{$node["Función: Preparar Contexto"].json.clientId}}/handovers`
- **Headers**: Autenticación apropiada
- **Body**:
```json
{
  "handoverId": "{{$node["Función: Preparar Contexto"].json.handoverId}}",
  "advisorId": "{{$node["Función: Preparar Contexto"].json.assignedAdvisor.id}}",
  "advisorName": "{{$node["Función: Preparar Contexto"].json.assignedAdvisor.name}}",
  "reason": "{{$node["Función: Preparar Contexto"].json.reason}}",
  "channelType": "{{$node["Función: Preparar Contexto"].json.channelType}}",
  "timestamp": "{{$node["Función: Preparar Contexto"].json.timestamp}}",
  "status": "pending"
}
```

#### Función: Generar Respuesta Transitoria
```javascript
// Generar una respuesta transitoria para el cliente
const input = items[0].json;
const channelType = $node["Función: Preparar Contexto"].json.channelType;
const advisorName = $node["Función: Preparar Contexto"].json.assignedAdvisor.name;

let response = "";
if (channelType === "whatsapp") {
  response = `Gracias por su interés en Selvadentro Tulum. Para brindarle la atención personalizada que merece, le estoy conectando con ${advisorName}, uno de nuestros asesores expertos, quien continuará asistiéndole. ${advisorName} se pondrá en contacto con usted en breve.`;
} else if (channelType === "voice") {
  response = `Gracias por su interés en Selvadentro Tulum. Para brindarle la mejor atención, le transferiré con ${advisorName}, uno de nuestros asesores expertos, quien podrá proporcionarle información más detallada y responder a todas sus preguntas. Por favor, espere un momento mientras realizamos la transferencia.`;
}

return {
  json: {
    handoverId: $node["Función: Preparar Contexto"].json.handoverId,
    transitionalResponse: response,
    advisorName: advisorName,
    channelType: channelType
  }
};
```

## Implementación y Despliegue

Para implementar estos workflows en n8n, sigue estos pasos:

1. **Configuración del Entorno n8n**:
   - Instala n8n siguiendo las [instrucciones oficiales](https://docs.n8n.io/hosting/)
   - Configura las variables de entorno necesarias (API keys, URLs, etc.)

2. **Importación de Workflows**:
   - Crea cada workflow en la interfaz de n8n
   - Configura los nodos según las especificaciones anteriores
   - Prueba cada workflow individualmente

3. **Integración con Sistemas Externos**:
   - Configura la integración con WhatsApp Business API
   - Configura la integración con Vapi
   - Configura la base de datos vectorial (Pinecone, Weaviate, etc.)
   - Configura la integración con el CRM

4. **Pruebas y Monitoreo**:
   - Realiza pruebas exhaustivas de cada workflow
   - Configura alertas para errores o fallos
   - Implementa un dashboard para monitorear el rendimiento

## Consideraciones Adicionales

1. **Seguridad y Privacidad**:
   - Implementa autenticación y autorización en todos los webhooks
   - Asegura el cifrado de datos sensibles
   - Cumple con regulaciones de privacidad (GDPR, LFPDPPP, etc.)

2. **Escalabilidad**:
   - Configura n8n para manejar múltiples instancias si es necesario
   - Implementa colas para manejar picos de tráfico
   - Considera el uso de caché para consultas frecuentes

3. **Mantenimiento**:
   - Programa actualizaciones regulares de la base de conocimiento
   - Revisa y optimiza los workflows periódicamente
   - Mantén actualizadas las integraciones con APIs externas

4. **Capacitación del Equipo**:
   - Capacita al equipo de ventas sobre cómo interactuar con el sistema
   - Proporciona documentación clara sobre los procesos de handover
   - Establece protocolos para la retroalimentación y mejora continua

## Conclusión

La implementación del Squad Colaborativo de Agentes de IA para Selvadentro Tulum en n8n proporciona una solución robusta, flexible y escalable. Los workflows diseñados permiten la orquestación efectiva de los diferentes agentes especializados, la integración con sistemas externos, y el aprendizaje continuo basado en datos reales.

Esta arquitectura no solo mejora la eficiencia operativa sino que crea una experiencia de cliente excepcional, combinando la disponibilidad 24/7 y la consistencia de la IA con el toque humano esencial para el mercado inmobiliario de lujo.