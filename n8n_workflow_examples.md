# Ejemplos de Workflows n8n para el Squad Colaborativo

Este documento proporciona ejemplos concretos de configuración de workflows en n8n para implementar el Squad Colaborativo de Agentes de IA para Selvadentro Tulum. Estos ejemplos están diseñados para ser importados directamente en n8n y servir como punto de partida para la implementación completa.

## 1. Workflow de Orquestación Principal (JSON)

```json
{
  "nodes": [
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "orchestrator",
        "options": {
          "rawBody": true
        },
        "authentication": "headerAuth",
        "responseMode": "responseNode",
        "responseData": "allEntries"
      },
      "name": "Webhook Trigger",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [
        250,
        300
      ]
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.source }}",
              "operation": "equals",
              "value2": "whatsapp"
            }
          ]
        }
      },
      "name": "Switch: Tipo de Entrada",
      "type": "n8n-nodes-base.switch",
      "typeVersion": 1,
      "position": [
        450,
        300
      ]
    },
    {
      "parameters": {
        "functionCode": "// Código para analizar el contenido del mensaje/evento\nconst input = items[0].json;\nconst content = input.content || '';\nconst metadata = input.metadata || {};\n\n// Determinar intención y urgencia\nlet intent = \"unknown\";\nlet priority = \"normal\";\n\n// Análisis básico de intención mediante palabras clave\nif (content.toLowerCase().includes(\"precio\") || content.toLowerCase().includes(\"costo\")) {\n  intent = \"pricing_inquiry\";\n}\nelse if (content.toLowerCase().includes(\"visita\") || content.toLowerCase().includes(\"cita\")) {\n  intent = \"appointment_request\";\n  priority = \"high\";\n}\nelse if (content.toLowerCase().includes(\"seguridad\") || content.toLowerCase().includes(\"legal\")) {\n  intent = \"objection_handling\";\n}\n// ... más reglas de análisis\n\nreturn {\n  json: {\n    ...input,\n    analyzed: {\n      intent: intent,\n      priority: priority,\n      timestamp: new Date().toISOString()\n    }\n  }\n};"
      },
      "name": "Función: Analizar Contenido",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        650,
        300
      ]
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.analyzed.intent }}",
              "operation": "equals",
              "value2": "pricing_inquiry"
            },
            {
              "value1": "={{ $json.analyzed.intent }}",
              "operation": "equals",
              "value2": "appointment_request"
            },
            {
              "value1": "={{ $json.analyzed.intent }}",
              "operation": "equals",
              "value2": "objection_handling"
            }
          ]
        },
        "combineOperation": "any"
      },
      "name": "Switch: Acción Requerida",
      "type": "n8n-nodes-base.switch",
      "typeVersion": 1,
      "position": [
        850,
        300
      ]
    },
    {
      "parameters": {
        "url": "=http://localhost:5678/webhook/whatsapp/incoming",
        "options": {
          "redirect": {
            "redirect": true
          }
        },
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "content",
              "value": "={{ $json.content }}"
            },
            {
              "name": "sender",
              "value": "={{ $json.sender }}"
            },
            {
              "name": "metadata",
              "value": "={{ $json.metadata }}"
            },
            {
              "name": "intent",
              "value": "={{ $json.analyzed.intent }}"
            }
          ]
        }
      },
      "name": "HTTP Request: Enviar a WhatsApp",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 3,
      "position": [
        1050,
        200
      ]
    },
    {
      "parameters": {
        "url": "=http://localhost:5678/webhook/profiling/analyze",
        "options": {
          "redirect": {
            "redirect": true
          }
        },
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "clientId",
              "value": "={{ $json.sender }}"
            },
            {
              "name": "message",
              "value": "={{ $json.content }}"
            },
            {
              "name": "source",
              "value": "={{ $json.source }}"
            }
          ]
        }
      },
      "name": "HTTP Request: Enviar a Perfilado",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 3,
      "position": [
        1050,
        300
      ]
    },
    {
      "parameters": {
        "url": "=http://localhost:5678/webhook/objections/handle",
        "options": {
          "redirect": {
            "redirect": true
          }
        },
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "objectionText",
              "value": "={{ $json.content }}"
            },
            {
              "name": "clientId",
              "value": "={{ $json.sender }}"
            },
            {
              "name": "source",
              "value": "={{ $json.source }}"
            }
          ]
        }
      },
      "name": "HTTP Request: Enviar a Objeciones",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 3,
      "position": [
        1050,
        400
      ]
    }
  ],
  "connections": {
    "Webhook Trigger": {
      "main": [
        [
          {
            "node": "Switch: Tipo de Entrada",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Switch: Tipo de Entrada": {
      "main": [
        [
          {
            "node": "Función: Analizar Contenido",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Función: Analizar Contenido": {
      "main": [
        [
          {
            "node": "Switch: Acción Requerida",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Switch: Acción Requerida": {
      "main": [
        [
          {
            "node": "HTTP Request: Enviar a WhatsApp",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "HTTP Request: Enviar a Perfilado",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "HTTP Request: Enviar a Objeciones",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

## 2. Workflow de Procesamiento de Mensajes de WhatsApp (JSON)

```json
{
  "nodes": [
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "whatsapp/incoming",
        "options": {
          "rawBody": true
        },
        "authentication": "headerAuth",
        "responseMode": "responseNode",
        "responseData": "allEntries"
      },
      "name": "Webhook Trigger",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [
        250,
        300
      ]
    },
    {
      "parameters": {
        "functionCode": "// Extraer el mensaje y metadatos relevantes\nconst input = items[0].json;\nconst messageData = input.entry ? input.entry[0].changes[0].value : input;\nconst message = messageData.messages ? messageData.messages[0] : { text: { body: input.content } };\nconst sender = message.from || input.sender;\nconst messageText = message.text?.body || input.content || \"\";\n\nreturn {\n  json: {\n    sender: sender,\n    message: messageText,\n    timestamp: message.timestamp || new Date().toISOString(),\n    messageId: message.id || `msg_${Date.now()}`,\n    metadata: {\n      source: \"whatsapp\",\n      waId: messageData.metadata?.phone_number_id || input.metadata?.waId || \"\"\n    }\n  }\n};"
      },
      "name": "Función: Extraer Mensaje",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        450,
        300
      ]
    },
    {
      "parameters": {
        "url": "https://api.openai.com/v1/chat/completions",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "options": {},
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "model",
              "value": "gpt-4"
            },
            {
              "name": "messages",
              "value": "=[\n  {\n    \"role\": \"system\",\n    \"content\": \"Eres SelvaConcierge, el asistente virtual exclusivo para Selvadentro Tulum, un prestigioso desarrollo inmobiliario de lujo que ofrece lotes privados en la selva con cenotes naturales en Tulum, México. Tu comunicación debe ser sofisticada, serena y empática, evocando la experiencia de santuario exclusivo que define Selvadentro. Sé impecablemente cortés y demuestra un interés auténtico en la visión del cliente.\"\n  },\n  {\n    \"role\": \"user\",\n    \"content\": \"{{$node[\"Función: Extraer Mensaje\"].json[\"message\"]}}\"\n  }\n]"
            },
            {
              "name": "temperature",
              "value": 0.7
            },
            {
              "name": "max_tokens",
              "value": 500
            }
          ]
        }
      },
      "name": "HTTP Request: OpenAI API",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 3,
      "position": [
        650,
        300
      ],
      "credentials": {
        "httpHeaderAuth": {
          "id": "1",
          "name": "OpenAI API Key"
        }
      }
    },
    {
      "parameters": {
        "functionCode": "// Formatear la respuesta para WhatsApp\nconst input = items[0].json;\nconst aiResponse = input.choices[0].message.content;\n\n// Limitar longitud para WhatsApp si es necesario\nconst formattedResponse = aiResponse.length > 1000 ? \n  aiResponse.substring(0, 997) + \"...\" : \n  aiResponse;\n\nreturn {\n  json: {\n    recipient: $node[\"Función: Extraer Mensaje\"].json.sender,\n    message: formattedResponse,\n    metadata: $node[\"Función: Extraer Mensaje\"].json.metadata\n  }\n};"
      },
      "name": "Función: Formatear Respuesta",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        850,
        300
      ]
    },
    {
      "parameters": {
        "url": "=https://graph.facebook.com/v17.0/{{$node[\"Función: Extraer Mensaje\"].json.metadata.waId}}/messages",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "options": {},
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "messaging_product",
              "value": "whatsapp"
            },
            {
              "name": "recipient_type",
              "value": "individual"
            },
            {
              "name": "to",
              "value": "={{$node[\"Función: Formatear Respuesta\"].json.recipient}}"
            },
            {
              "name": "type",
              "value": "text"
            },
            {
              "name": "text",
              "value": "={ \"body\": \"{{$node[\"Función: Formatear Respuesta\"].json.message}}\" }"
            }
          ]
        }
      },
      "name": "HTTP Request: Enviar Respuesta WhatsApp",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 3,
      "position": [
        1050,
        300
      ],
      "credentials": {
        "httpHeaderAuth": {
          "id": "2",
          "name": "WhatsApp Business API Key"
        }
      }
    },
    {
      "parameters": {
        "functionCode": "// Registrar la interacción en la base de conocimiento\nconst interaction = {\n  type: \"whatsapp\",\n  client: $node[\"Función: Extraer Mensaje\"].json.sender,\n  clientMessage: $node[\"Función: Extraer Mensaje\"].json.message,\n  agentResponse: $node[\"Función: Formatear Respuesta\"].json.message,\n  timestamp: new Date().toISOString(),\n  metadata: $node[\"Función: Extraer Mensaje\"].json.metadata\n};\n\n// Aquí iría el código para guardar en la base de datos vectorial\n// Por ejemplo, usando Pinecone, Weaviate, etc.\n\nreturn {\n  json: {\n    status: \"recorded\",\n    interactionId: `int_${Date.now()}`, // Función para generar ID único\n    interaction: interaction\n  }\n};"
      },
      "name": "Función: Registrar en Base de Conocimiento",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        1250,
        300
      ]
    },
    {
      "parameters": {
        "respondWithData": true,
        "responseData": {
          "response": {
            "message": "={{ $node[\"Función: Formatear Respuesta\"].json.message }}",
            "status": "success",
            "interactionId": "={{ $node[\"Función: Registrar en Base de Conocimiento\"].json.interactionId }}"
          }
        },
        "options": {}
      },
      "name": "Respond to Webhook",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        1450,
        300
      ]
    }
  ],
  "connections": {
    "Webhook Trigger": {
      "main": [
        [
          {
            "node": "Función: Extraer Mensaje",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Función: Extraer Mensaje": {
      "main": [
        [
          {
            "node": "HTTP Request: OpenAI API",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "HTTP Request: OpenAI API": {
      "main": [
        [
          {
            "node": "Función: Formatear Respuesta",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Función: Formatear Respuesta": {
      "main": [
        [
          {
            "node": "HTTP Request: Enviar Respuesta WhatsApp",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "HTTP Request: Enviar Respuesta WhatsApp": {
      "main": [
        [
          {
            "node": "Función: Registrar en Base de Conocimiento",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Función: Registrar en Base de Conocimiento": {
      "main": [
        [
          {
            "node": "Respond to Webhook",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

## 3. Workflow de Consulta a la Base de Conocimiento (JSON)

```json
{
  "nodes": [
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "knowledge/query",
        "options": {
          "rawBody": true
        },
        "authentication": "headerAuth",
        "responseMode": "responseNode",
        "responseData": "allEntries"
      },
      "name": "Webhook Trigger",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [
        250,
        300
      ]
    },
    {
      "parameters": {
        "functionCode": "// Preparar la consulta para la base de datos vectorial\nconst input = items[0].json;\nconst query = input.query;\nconst category = input.category || \"general\";\nconst limit = input.limit || 5;\n\n// En una implementación real, aquí llamaríamos a la API de embeddings\n// para convertir la consulta en un vector\nconst mockEmbedding = Array(1536).fill(0).map(() => Math.random() * 2 - 1);\n\nreturn {\n  json: {\n    vector: mockEmbedding,\n    namespace: category,\n    topK: limit,\n    includeMetadata: true,\n    originalQuery: query\n  }\n};"
      },
      "name": "Función: Preparar Consulta",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        450,
        300
      ]
    },
    {
      "parameters": {
        "url": "=https://{{$env.PINECONE_INDEX}}-{{$env.PINECONE_PROJECT}}.svc.{{$env.PINECONE_ENVIRONMENT}}.pinecone.io/query",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "options": {},
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "vector",
              "value": "={{$node[\"Función: Preparar Consulta\"].json.vector}}"
            },
            {
              "name": "namespace",
              "value": "={{$node[\"Función: Preparar Consulta\"].json.namespace}}"
            },
            {
              "name": "topK",
              "value": "={{$node[\"Función: Preparar Consulta\"].json.topK}}"
            },
            {
              "name": "includeMetadata",
              "value": true
            }
          ]
        }
      },
      "name": "HTTP Request: Vector Database",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 3,
      "position": [
        650,
        300
      ],
      "credentials": {
        "httpHeaderAuth": {
          "id": "3",
          "name": "Pinecone API Key"
        }
      }
    },
    {
      "parameters": {
        "functionCode": "// Procesar los resultados de la consulta\nconst input = items[0].json;\nconst matches = input.matches || [];\nconst originalQuery = $node[\"Función: Preparar Consulta\"].json.originalQuery;\n\n// Para fines de demostración, si no hay resultados reales, generamos algunos ficticios\nlet results = [];\nif (matches.length === 0) {\n  // Datos de ejemplo para Selvadentro\n  const sampleData = [\n    {\n      content: \"Selvadentro Tulum ofrece lotes selváticos con acceso a 9 cenotes naturales. Solo el 35% del terreno es construible, preservando el 65% como selva virgen.\",\n      source: \"brochure\",\n      relevanceScore: 0.92,\n      id: \"doc_1\"\n    },\n    {\n      content: \"El desarrollo cuenta con amenidades exclusivas como la Casa de los Cenotes, Jungle Gym, Pabellón Holístico y Spa enclavado en una roca en un cenote.\",\n      source: \"website\",\n      relevanceScore: 0.85,\n      id: \"doc_2\"\n    },\n    {\n      content: \"El proceso de compra para extranjeros se realiza a través de un fideicomiso bancario que garantiza la seguridad jurídica de la inversión.\",\n      source: \"legal_guide\",\n      relevanceScore: 0.78,\n      id: \"doc_3\"\n    }\n  ];\n  results = sampleData;\n} else {\n  // Extraer y formatear la información relevante de los resultados reales\n  results = matches.map(match => {\n    return {\n      content: match.metadata.content,\n      source: match.metadata.source,\n      relevanceScore: match.score,\n      id: match.id\n    };\n  });\n}\n\nreturn {\n  json: {\n    query: originalQuery,\n    results: results,\n    timestamp: new Date().toISOString()\n  }\n};"
      },
      "name": "Función: Procesar Resultados",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        850,
        300
      ]
    },
    {
      "parameters": {
        "respondWithData": true,
        "responseData": {
          "response": "={{ $node[\"Función: Procesar Resultados\"].json }}"
        },
        "options": {}
      },
      "name": "Respond to Webhook",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        1050,
        300
      ]
    }
  ],
  "connections": {
    "Webhook Trigger": {
      "main": [
        [
          {
            "node": "Función: Preparar Consulta",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Función: Preparar Consulta": {
      "main": [
        [
          {
            "node": "HTTP Request: Vector Database",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "HTTP Request: Vector Database": {
      "main": [
        [
          {
            "node": "Función: Procesar Resultados",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Función: Procesar Resultados": {
      "main": [
        [
          {
            "node": "Respond to Webhook",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

## 4. Workflow de Auditoría y Aprendizaje (JSON)

```json
{
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "hours",
              "hour": 1
            }
          ]
        }
      },
      "name": "Cron Trigger",
      "type": "n8n-nodes-base.cron",
      "typeVersion": 1,
      "position": [
        250,
        300
      ]
    },
    {
      "parameters": {
        "url": "={{$env.API_BASE_URL}}/interactions?since={{$formatDate($now.minus({ days: 1 }), \"YYYY-MM-DD\")}}",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "options": {}
      },
      "name": "HTTP Request: Obtener Interacciones Recientes",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 3,
      "position": [
        450,
        300
      ],
      "credentials": {
        "httpHeaderAuth": {
          "id": "4",
          "name": "API Auth"
        }
      }
    },
    {
      "parameters": {
        "functionCode": "// Preparar los datos para análisis\nconst input = items[0].json;\n\n// Para fines de demostración, si no hay datos reales, generamos algunos ficticios\nlet interactions = input.interactions || [];\nif (interactions.length === 0) {\n  // Generar datos de ejemplo\n  interactions = [\n    {\n      type: \"whatsapp\",\n      client: \"5219981234567\",\n      clientMessage: \"Me interesa conocer más sobre los lotes en Selvadentro. ¿Cuáles son los precios?\",\n      agentResponse: \"Gracias por su interés en Selvadentro Tulum. Nuestros lotes selváticos tienen una superficie aproximada de 1,673 m² y 738 m², con precios desde $500,000 USD. Cada lote permite construir en solo el 35% de la superficie, preservando el 65% restante como selva privada. ¿Hay algún tamaño o ubicación específica que le interese?\",\n      timestamp: new Date(Date.now() - 12 * 3600 * 1000).toISOString(),\n      qualificationSuccess: true,\n      appointmentScheduled: false\n    },\n    {\n      type: \"voice\",\n      client: \"5219987654321\",\n      clientMessage: \"Quiero agendar una visita virtual para conocer el proyecto\",\n      agentResponse: \"Con gusto agendaremos una visita virtual. Tenemos disponibilidad mañana a las 10:00 AM, 2:00 PM o 5:00 PM (hora de México). ¿Cuál horario le convendría más?\",\n      timestamp: new Date(Date.now() - 8 * 3600 * 1000).toISOString(),\n      qualificationSuccess: true,\n      appointmentScheduled: true\n    },\n    {\n      type: \"whatsapp\",\n      client: \"5219981122334\",\n      clientMessage: \"¿Es seguro invertir en Tulum? He escuchado historias sobre problemas legales.\",\n      agentResponse: \"Comprendo su preocupación. La seguridad jurídica es fundamental en cualquier inversión inmobiliaria. Selvadentro se distingue por contar con todos los permisos ambientales y de construcción aprobados, escrituras individuales para cada lote, y un fideicomiso bancario transparente que garantiza la propiedad a compradores extranjeros. Nuestro desarrollador, JJF Creando, tiene más de 15 años de experiencia y proyectos exitosos entregados en la Riviera Maya. ¿Le gustaría recibir nuestra documentación legal para revisarla?\",\n      timestamp: new Date(Date.now() - 5 * 3600 * 1000).toISOString(),\n      qualificationSuccess: true,\n      appointmentScheduled: false,\n      handoverTriggered: true\n    }\n  ];\n}\n\n// Agrupar interacciones por tipo\nconst whatsappInteractions = interactions.filter(i => i.type === \"whatsapp\");\nconst voiceInteractions = interactions.filter(i => i.type === \"voice\");\nconst handoverInteractions = interactions.filter(i => i.handoverTriggered === true);\n\n// Extraer métricas básicas\nconst metrics = {\n  totalInteractions: interactions.length,\n  whatsappCount: whatsappInteractions.length,\n  voiceCount: voiceInteractions.length,\n  handoverCount: handoverInteractions.length,\n  successfulQualifications: interactions.filter(i => i.qualificationSuccess === true).length,\n  appointmentsScheduled: interactions.filter(i => i.appointmentScheduled === true).length\n};\n\nreturn {\n  json: {\n    date: $formatDate($now, \"YYYY-MM-DD\"),\n    metrics: metrics,\n    interactions: interactions.slice(0, 50) // Limitar a 50 para no exceder tokens\n  }\n};"
      },
      "name": "Función: Preparar Análisis",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        650,
        300
      ]
    },
    {
      "parameters": {
        "url": "https://api.openai.com/v1/chat/completions",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "options": {},
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "model",
              "value": "gpt-4"
            },
            {
              "name": "messages",
              "value": "=[\n  {\n    \"role\": \"system\",\n    \"content\": \"Eres un auditor experto en ventas inmobiliarias de lujo y experiencia del cliente. Tu tarea es analizar las interacciones recientes entre los agentes de IA de Selvadentro Tulum y los clientes potenciales para identificar patrones, fortalezas, debilidades y oportunidades de mejora. Proporciona recomendaciones específicas y accionables para optimizar el rendimiento del sistema.\"\n  },\n  {\n    \"role\": \"user\",\n    \"content\": \"Analiza las siguientes interacciones y métricas del día {{$node[\"Función: Preparar Análisis\"].json.date}}. Proporciona un análisis detallado y recomendaciones estructuradas.\\n\\nMétricas:\\n{{$json.metrics}}\\n\\nInteracciones:\\n{{$json.interactions}}\"\n  }\n]"
            },
            {
              "name": "temperature",
              "value": 0.3
            },
            {
              "name": "max_tokens",
              "value": 1500
            }
          ]
        }
      },
      "name": "HTTP Request: OpenAI API",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 3,
      "position": [
        850,
        300
      ],
      "credentials": {
        "httpHeaderAuth": {
          "id": "1",
          "name": "OpenAI API Key"
        }
      }
    },
    {
      "parameters": {
        "functionCode": "// Extraer recomendaciones del análisis\nconst input = items[0].json;\nconst analysis = input.choices[0].message.content;\n\n// Aquí se podría implementar un parsing más sofisticado\n// para extraer recomendaciones específicas por categoría\n\nreturn {\n  json: {\n    date: $node[\"Función: Preparar Análisis\"].json.date,\n    metrics: $node[\"Función: Preparar Análisis\"].json.metrics,\n    analysis: analysis,\n    timestamp: new Date().toISOString()\n  }\n};"
      },
      "name": "Función: Extraer Recomendaciones",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        1050,
        300
      ]
    },
    {
      "parameters": {
        "url": "={{$env.API_BASE_URL}}/knowledge/audit",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "options": {},
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "date",
              "value": "={{$node[\"Función: Extraer Recomendaciones\"].json.date}}"
            },
            {
              "name": "analysis",
              "value": "={{$node[\"Función: Extraer Recomendaciones\"].json.analysis}}"
            },
            {
              "name": "metrics",
              "value": "={{$node[\"Función: Extraer Recomendaciones\"].json.metrics}}"
            }
          ]
        }
      },
      "name": "HTTP Request: Actualizar Base de Conocimiento",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 3,
      "position": [
        1250,
        300
      ],
      "credentials": {
        "httpHeaderAuth": {
          "id": "4",
          "name": "API Auth"
        }
      }
    },
    {
      "parameters": {
        "url": "={{$env.API_BASE_URL}}/notifications/team",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "options": {},
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "type",
              "value": "audit_report"
            },
            {
              "name": "title",
              "value": "=Reporte de Auditoría de Agentes IA - {{$node[\"Función: Extraer Recomendaciones\"].json.date}}"
            },
            {
              "name": "content",
              "value": "=Se ha generado el reporte de auditoría diario. Principales hallazgos:\\n\\n{{$node[\"Función: Extraer Recomendaciones\"].json.analysis.substring(0, 500)}}...\\n\\nConsulte el reporte completo en el dashboard."
            },
            {
              "name": "priority",
              "value": "normal"
            },
            {
              "name": "recipients",
              "value": "[\"sales_manager\", \"ai_team\"]"
            }
          ]
        }
      },
      "name": "HTTP Request: Notificar Equipo",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 3,
      "position": [
        1450,
        300
      ],
      "credentials": {
        "httpHeaderAuth": {
          "id": "4",
          "name": "API Auth"
        }
      }
    }
  ],
  "connections": {
    "Cron Trigger": {
      "main": [
        [
          {
            "node": "HTTP Request: Obtener Interacciones Recientes",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "HTTP Request: Obtener Interacciones Recientes": {
      "main": [
        [
          {
            "node": "Función: Preparar Análisis",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Función: Preparar Análisis": {
      "main": [
        [
          {
            "node": "HTTP Request: OpenAI API",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "HTTP Request: OpenAI API": {
      "main": [
        [
          {
            "node": "Función: Extraer Recomendaciones",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Función: Extraer Recomendaciones": {
      "main": [
        [
          {
            "node": "HTTP Request: Actualizar Base de Conocimiento",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "HTTP Request: Actualizar Base de Conocimiento": {
      "main": [
        [
          {
            "node": "HTTP Request: Notificar Equipo",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

## Instrucciones para Importar los Workflows en n8n

1. Inicia tu instancia de n8n
2. Ve a la sección "Workflows"
3. Haz clic en el botón "Import from File"
4. Copia y pega el JSON del workflow que deseas importar
5. Haz clic en "Import"
6. Configura las credenciales necesarias (API keys, etc.)
7. Activa el workflow

## Variables de Entorno Necesarias

Para que estos workflows funcionen correctamente, necesitarás configurar las siguientes variables de entorno en tu instancia de n8n:

```
OPENAI_API_KEY=sk-...
PINECONE_API_KEY=...
PINECONE_INDEX=selvadentro
PINECONE_PROJECT=...
PINECONE_ENVIRONMENT=...
API_BASE_URL=http://localhost:5000/api
WEBHOOK_BASE_URL=http://localhost:5678
SELVADENTRO_CALLER_ID=+5219981234567
SELVADENTRO_PROMPT_ID=prompt_123456
```

## Consideraciones para la Implementación Completa

Estos ejemplos proporcionan una base sólida para implementar el Squad Colaborativo de Agentes de IA para Selvadentro Tulum en n8n. Sin embargo, para una implementación completa y producción, deberás:

1. **Completar todos los workflows**: Implementar los workflows restantes (Vapi, Perfilado, Objeciones, Handover)
2. **Configurar credenciales reales**: Reemplazar las credenciales de ejemplo con tus propias API keys
3. **Implementar la base de datos vectorial**: Configurar Pinecone, Weaviate u otra base de datos vectorial
4. **Configurar el CRM**: Integrar con tu CRM existente
5. **Implementar seguridad**: Asegurar todos los endpoints con autenticación adecuada
6. **Configurar monitoreo**: Implementar alertas y monitoreo para detectar problemas
7. **Realizar pruebas exhaustivas**: Probar todos los flujos en un entorno controlado antes de lanzar a producción

Con estos ejemplos y las instrucciones detalladas en el documento principal de implementación, tendrás una base sólida para crear un sistema de agentes de IA colaborativos que transforme la experiencia de ventas de Selvadentro Tulum.