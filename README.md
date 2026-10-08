# ☕ Café AI — Agente de WhatsApp con Function Calling

Agente conversacional inteligente para cafeterías, construido con n8n y OpenAI. Atiende pedidos, gestiona reservaciones, consulta el menú y lanza campañas de retención automáticas, todo desde WhatsApp Business.

---

## 🧩 El problema que resuelve

Una cafetería con poco personal recibía pedidos y consultas por WhatsApp manualmente: alguien tenía que estar pendiente del teléfono, responder preguntas del menú, tomar reservas, coordinar con cocina y hacer seguimiento a clientes inactivos. Era lento, generaba errores y dependía totalmente de disponibilidad humana.

**Café AI automatiza el 100% de esa operación.** El bot atiende 24/7, procesa pagos, notifica a cocina y recupera clientes sin intervención manual.

---

## 🏗️ Arquitectura

El sistema está compuesto por **4 flujos independientes** que corren en paralelo:

```
┌─────────────────────────────────────────────────────────────┐
│                        CAFÉ AI                              │
│                                                             │
│  [1] AGENTE CONVERSACIONAL (WhatsApp entrante)             │
│      Webhook ──► Normalizar ──► Construir Mensajes         │
│      ──► OpenAI 1ra Llamada ──► ¿Requiere Herramienta?    │
│                                    │                        │
│            NO ◄────────────────────┤                       │
│           Enviar respuesta         │ SÍ                     │
│                                    ▼                        │
│                         Enrutador de Herramienta            │
│                         ┌──────────────────────┐           │
│                         │ • verificar_disp.    │           │
│                         │ • crear_reserva      │           │
│                         │ • consultar_menu     │           │
│                         │ • crear_pedido       │──► Pago  │
│                         │ • consultar_estado   │           │
│                         │ • registrar_feedback │──► Alerta│
│                         └──────────────────────┘           │
│                                    │                        │
│                    OpenAI 2da Llamada ──► Enviar WhatsApp  │
│                                                             │
│  [2] CONFIRMACIÓN DE PAGO                                   │
│      Webhook pago ──► Confirmar pedido ──► Notificar cocina│
│                                                             │
│  [3] PEDIDO COMPLETADO                                      │
│      Webhook completado ──► Enviar solicitud de feedback   │
│                                                             │
│  [4] RETENCIÓN DIARIA                                       │
│      Cron 10:00 AM ──► Obtener inactivos (30 días)        │
│      ──► Campaña WhatsApp personalizada                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Stack tecnológico

| Componente | Tecnología |
|---|---|
| Orquestación de flujos | n8n |
| Modelo de lenguaje | OpenAI GPT-4o mini |
| Patrón de herramientas | OpenAI Function Calling (dos llamadas) |
| Canal de mensajería | WhatsApp Business API (graph.facebook.com) |
| Backend del negocio | API REST propia (`CAFE_API_BASE_URL`) |
| Activación programada | Cron trigger (retención diaria) |

---

## 🤖 Patrón de dos llamadas (Function Calling)

El agente usa el patrón oficial de OpenAI Function Calling para decidir cuándo ejecutar una acción vs. responder directamente:

```
Usuario envía mensaje
        │
        ▼
[1ra Llamada OpenAI]
  Modelo recibe: historial + mensaje + definiciones de herramientas
  Modelo responde: "necesito llamar a verificar_disponibilidad"
  o bien: texto directo (sin herramienta)
        │
    ¿tool_call?
   NO ──┼── SÍ
   │    │    │
   │    │    ▼
   │    │  Ejecutar HTTP request a CAFE_API_BASE_URL
   │    │  Recibir resultado de la herramienta
   │    │    │
   │    │    ▼
   │    │  [2da Llamada OpenAI]
   │    │  Modelo recibe: historial + resultado de herramienta
   │    │  Modelo genera: respuesta final en lenguaje natural
   │    │    │
   └────┴────┘
        │
        ▼
   Enviar mensaje por WhatsApp
```

Esto permite que el bot sea **completamente conversacional** sin reglas if/else manuales.

---

## 🔧 Las 6 herramientas disponibles

| Herramienta | Qué hace |
|---|---|
| `verificar_disponibilidad` | Consulta horarios y mesas disponibles para una fecha/hora |
| `crear_reserva` | Registra una reservación con nombre, fecha, hora y número de personas |
| `consultar_menu` | Devuelve el menú completo o filtrado por categoría |
| `crear_pedido` | Crea un pedido, genera cotización y enlace de pago |
| `consultar_estado_pedido` | Devuelve el estado actual de un pedido (preparando, listo, entregado) |
| `registrar_feedback` | Guarda la calificación del cliente; si es baja, alerta al gerente automáticamente |

---

## ⚙️ Configuración

### Variables de entorno requeridas

Configura estas credenciales en n8n → Settings → Credentials:

| Variable | Descripción |
|---|---|
| `OPENAI_API_KEY` | API Key de OpenAI (plataforma.openai.com) |
| `WHATSAPP_TOKEN` | Token de acceso de WhatsApp Business API |
| `WHATSAPP_PHONE_NUMBER_ID` | ID del número de WhatsApp registrado en Meta |
| `CAFE_API_BASE_URL` | URL base de tu backend (ej. `https://api.micafe.mx`) |
| `MANAGER_WHATSAPP` | Número del gerente para alertas de feedback negativo |

### Endpoints esperados en `CAFE_API_BASE_URL`

El flujo llama a estos endpoints REST de tu sistema:

```
GET  /disponibilidad?fecha=&hora=&personas=
POST /reservas
GET  /menu?categoria=
POST /pedidos
GET  /pedidos/{id}/estado
POST /feedback
GET  /clientes/inactivos?dias=30
```

---

## 🚀 Instalación

1. **Clona el repositorio**
   ```bash
   git clone https://github.com/Jairh1294/Cafe-AI.git
   ```

2. **Importa el workflow en n8n**
   - Abre tu instancia de n8n
   - Ve a **Workflows → Import from File**
   - Selecciona `cafe_ai_openai_workflow.json`

3. **Configura las credenciales**
   - En n8n: **Settings → Credentials → New**
   - Agrega `OpenAI API`, `HTTP Header Auth` para WhatsApp

4. **Configura los webhooks en Meta**
   - En [Meta for Developers](https://developers.facebook.com/), configura:
     - Webhook de mensajes entrantes → URL del nodo **WHATSAPP — Entrante**
     - Webhook de pagos → URL del nodo **PAGO**
     - Webhook de pedidos completados → URL del nodo **PEDIDO COMPLETADO**

5. **Activa el workflow**
   - Haz clic en **Activate** en la parte superior de n8n
   - El cron de retención se ejecutará automáticamente a las 10:00 AM

---

## 💬 Ejemplo de conversación

```
👤 Cliente: Hola, ¿tienen mesa para 2 personas esta noche a las 8?

🤖 Bot: ¡Hola! Déjame verificar disponibilidad...
        [verifica disponibilidad → API devuelve 3 opciones]
        Sí, tenemos mesa disponible a las 8:00 PM para 2 personas.
        ¿Quieres hacer la reservación? Solo necesito tu nombre. 😊

👤 Cliente: Sí, a nombre de Carlos

🤖 Bot: [crea reserva → API confirma]
        ¡Listo, Carlos! Tu reservación está confirmada:
        📅 Hoy a las 8:00 PM | 👥 2 personas
        Te esperamos. ¿Algo más en lo que pueda ayudarte?
```

---

## 📊 Flujo de pedido con pago

```
Cliente solicita pedido
        │
        ▼
crear_pedido → Cotizar pedido
        │
        ▼
Generar enlace de pago (Stripe/MercadoPago)
        │
        ▼
Bot envía enlace por WhatsApp
        │
        ▼
Cliente paga → Webhook PAGO recibido
        │
        ▼
Confirmar pedido + Notificar a cocina
        │
        ▼
Cocina completa → Webhook PEDIDO COMPLETADO
        │
        ▼
Bot pide feedback al cliente
        │
   Calificación baja?
   SÍ → Alerta automática al gerente por WhatsApp
```

---

## 🔄 Campaña de retención diaria

Cada día a las 10:00 AM, el sistema:
1. Consulta clientes que no han visitado en los últimos **30 días**
2. Genera un mensaje personalizado para cada uno
3. Lo envía por WhatsApp automáticamente

Esto recupera clientes inactivos sin esfuerzo manual.

---

## 📄 Licencia

MIT License — libre para uso comercial y personal.

---

*Construido con n8n + OpenAI GPT-4o mini · WhatsApp Business API*
