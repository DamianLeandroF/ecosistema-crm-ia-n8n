# Ecosistema Autónomo de Calificación de Leads y Generación de Propuestas (CRM + IA)

Ecosistema de automatización empresarial desarrollado sobre **n8n**, diseñado para la ingesta, evaluación inteligente, persistencia y despacho de propuestas comerciales para leads entrantes. Integra **Airtable** como sistema de registro (CRM/SSOT), modelos LLM de **Google Gemini** para análisis predictivo y redacción de propuestas personalizadas, un punto de control humano (**Human-in-the-Loop - HITL**) y **Gmail** como canal de salida.

---

## 🏗️ Arquitectura del Sistema

El flujo implementa una arquitectura modular con bifurcación de contingencia y persistencia de memoria intermedia:

```text
[Airtable Trigger] (Lead en 'Pendiente')
        │
        ▼
[Google Gemini LLM] (Scoring BANT/LTV + Redacción Propuesta)
   │           │
(Success)   (Error) ──► [Airtable Error Handler] (Estado: 'Error')
   │
   ▼
[Airtable Persistence] (Guarda Propuesta + Estado: 'Esperando Aprobación')
   │
   ▼
[Wait Node - HITL] (Pausa dinámica vía Webhook GET / URL de Reanudación)
   │ (Validación / Clic Humano)
   ▼
[Gmail Delivery] (Envío automático de propuesta al cliente)
   │
   ▼
[Airtable Final Update] (Cierre de ciclo: Estado 'Propuesta Enviada')
```

---

## ⚙️ Componentes y Tecnologías

| Componente | Rol en la Solución | Configuración Clave |
| :--- | :--- | :--- |
| **Airtable** | CRM / Base de datos relacional | Filtro poller: `{Estado} = 'Pendiente'`. Columnas dinámicas mapeadas por `fields`. |
| **Google Gemini (LLM)** | Motor cognitivo / Decisor | Prompt estructurado para análisis de viabilidad, cálculo de Score (0-100) y propuesta B2B. |
| **Airtable (Persistencia)** | Memoria intermedia del flujo | Almacena `Score_IA`, `Propuesta_Generada` y transiciona estado a `Esperando Aprobación`. |
| **n8n Wait Node (HITL)** | Gobernanza humana de seguridad | Pausa nativa con reactivación mediante URL dinámica (`$execution.resumeUrl` vía HTTP GET). |
| **Gmail** | Canal de salida al cliente | Mapeo dinámico de destinatario (`{{ fields.Email }}`) y cuerpo (`{{ fields.Propuesta_Generada }}`). |
| **Error Handling Branch** | Resiliencia operativa | Enrutamiento condicional (`Continue using error output`) para aislar fallos de API. |

---

## 🛡️ Gobernanza: Human-in-the-Loop (HITL)

Para evitar alucinaciones, errores de tono o cotizaciones erróneas hacia clientes reales, el sistema no despacha correos de manera desatendida:

1. El nodo **Wait** detiene la ejecución inmediatamente después de actualizar la base de datos con la propuesta generada.
2. Se genera una dirección web de reanudación única por cada ejecución.
3. El revisor humano inspecciona la propuesta en Airtable o mediante el enlace de validación.
4. Al abrir la URL de reanudación vía navegador (petición GET), el flujo se desbloquea en tiempo real y procede al despacho por Gmail.

---

## 🚨 Manejo de Errores y Tolerancia a Fallos

El nodo de Gemini cuenta con la directiva **`Continue (using error output)`**:
* **Ruta de Éxito (`Success`):** El flujo continúa hacia la persistencia y la cola de espera de aprobación.
* **Ruta de Contingencia (`Error`):** Si la cuota de la API se satura, el JSON de entrada está truncado o el LLM no responde, el flujo deriva a un nodo alternativo de Airtable que marca el registro con `Estado = 'Error'`, evitando ejecuciones congeladas o fallos silenciosos.

---


## 🚀 Guía de Despliegue

### 1. Requisitos Previos
* Instancia activa de **n8n** (Cloud o Self-hosted).
* Cuenta de **Airtable** con base de datos configurada.
* API Key de **Google Gemini** activa.
* Credencial OAuth2 o App Password de **Google / Gmail** conectada a n8n.

### 2. Configuración en Airtable
Crear una tabla llamada `Leads_Propuestas` con las siguientes columnas exactas:
* `Nombre_Cliente` (Single line text)
* `Email` (Email)
* `Empresa` (Single line text)
* `Descripcion_Proyecto` (Long text)
* `Estado` (Single select: `Pendiente`, `Esperando Aprobación`, `Propuesta Enviada`, `Error`)
* `Score_IA` (Number)
* `Propuesta_Generada` (Long text)
* `Fecha_Creacion` (Created time)

### 3. Importación en n8n
1. Descarga el archivo `workflows/ecosistema_ia.json`.
2. En tu panel de n8n, crea un nuevo workflow, abre el menú de opciones (tres puntos superiores) y selecciona **Import from File**.
3. Abre cada nodo de Airtable y selecciona tu credencial y tu Base ID.
4. Abre el nodo de IA y vincula tu API Key de Gemini.
5. Abre el nodo de Gmail y selecciona tu cuenta conectada.
6. Guarda y activa el workflow (**Publish**).

---

## 🔗 Enlaces del Proyecto

* **Base de Airtable (Modo Lectura):** `https://airtable.com/appXQtUw8TQemu5dK/shrqaGe2sEbG98iyr`
