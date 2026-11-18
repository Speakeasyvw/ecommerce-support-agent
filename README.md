# Ecommerce Support Agent

Agente de atención al cliente para e-commerce construido con LangChain y Azure OpenAI.
Lee tickets de clientes, identifica el pedido, consulta la base de órdenes y redacta una
respuesta lista para enviar. Si no encuentra información suficiente, escala el caso a un humano.

> Todos los datos incluidos en `data/` (tickets, clientes, pedidos, tiendas y trackings) son **ficticios**
> y existen solo para demostrar el funcionamiento.

## Cómo funciona

```
Ticket (.txt / API / interfaz)
        │
        ▼
 Agente ReAct (LangChain + Azure OpenAI)
        │  decide qué herramienta usar
        ├── BuscarPedidoPorID    → busca por número de orden en el Excel
        ├── BuscarPedidoPorMail  → si no hay número, busca por email (tolerante a puntos/mayúsculas)
        ├── GenerarRespuesta     → redacta la respuesta con los datos reales del pedido
        └── EscalarAHumano       → si no hay datos suficientes
        │
        ▼
 Respuesta al cliente (firmada como "Argo")
```

Componentes:

| Módulo | Función |
|---|---|
| `agent/loader.py` | Lee los tickets de `data/tickets/` y extrae remitente, asunto y cuerpo. |
| `agent/database.py` | Consulta de pedidos sobre `data/master_flee.xlsx` y extracción de order ID / email. |
| `agent/classifier.py` | Clasifica la intención del ticket (envío, tracking, reembolso, daño, etc.). |
| `agent/responder.py` | Genera la respuesta y agrega siempre los datos clave (tracking, producto, fecha). |
| `agent/historial.py` | Guarda cada interacción en SQLite para dar contexto a consultas posteriores. |
| `api.py` | API REST con FastAPI (`POST /responder`). |
| `frontend.py` | Interfaz web en Streamlit que consume la API. |

Principio de diseño: el modelo **no inventa datos del pedido**. Toda la información (estado,
tracking, fechas) sale de la base; si no se encuentra, el caso se escala.

## Instalación

Requiere Python 3.10+ y un recurso de Azure OpenAI.

```bash
python -m venv env
env\Scripts\activate          # Windows
# source env/bin/activate     # Linux / macOS
pip install -r requirements.txt
```

> En Windows, si PowerShell bloquea la activación del entorno, ejecutá
> `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser` y volvé a activarlo.

Copiá `.env.example` como `.env` y completá tus credenciales:

```
AZURE_API_KEY=...
AZURE_ENDPOINT=https://<tu-recurso>.openai.azure.com/
AZURE_DEPLOYMENT_NAME=...
AZURE_API_VERSION=...
```

## Uso

**Procesar tickets de ejemplo** (los primeros 3 de `data/tickets/`):

```bash
python main.py
```

**API REST:**

```bash
uvicorn api:app --reload
```

```bash
curl -X POST http://localhost:8000/responder \
  -H "Content-Type: application/json" \
  -d '{"ticket": "From: laura.mendez@example.com\nSubject: Question about my order\nOrder: 100101\n\"When will my order ship?\""}'
```

**Interfaz web** (con la API levantada en otra terminal):

```bash
streamlit run frontend.py
```

## Tests

```bash
pytest tests/
```

`test_clasificar_ticket` llama al modelo, por lo que necesita las credenciales de Azure configuradas.

## Datos de ejemplo

`data/master_flee.xlsx` contiene 12 pedidos ficticios de dos tiendas inventadas, con distintos
estados (en proceso, en tránsito, tracking sin actualizar, reembolso, producto dañado, dirección
incorrecta, producto equivocado, retenido en aduana). Los 15 tickets de `data/tickets/` cubren esos
casos e incluyen consultas sin número de orden (búsqueda por email) y pedidos inexistentes (escalamiento).

## Próximas mejoras

- Integración multicanal (email, WhatsApp).
- Contexto con políticas de la empresa (devoluciones, plazos) vía RAG.
- Notificación automática al equipo cuando un caso se escala.
- Panel de administración con métricas de respuestas y escalamientos.
