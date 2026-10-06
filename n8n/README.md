# ⚡ Demostración de Automatización de Eventos con n8n

## 🎯 Objetivo

Configurar un flujo de trabajo **asíncrono e interactivo en n8n** que permita:

- Recibir una solicitud mediante un **Webhook HTTP POST**.
- Procesar los datos recibidos.
- Ejecutar reglas de negocio mediante código **JavaScript**.
- Determinar automáticamente el resultado de una solicitud.
- Enrutar el flujo según el resultado de la evaluación.
- Almacenar información en una base de datos.
- Enviar una notificación por correo electrónico cuando sea necesario.
- Responder al sistema que realizó la solicitud.

---

# 📐 Estructura del Flujo

El flujo de automatización está compuesto por los siguientes nodos:

```text
┌──────────────────┐
│   Webhook POST   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   Code (JS)      │
│ Reglas de negocio│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   IF / Switch    │
└───────┬──────────┘
        │
    ┌───┴───────────────┐
    │                   │
   TRUE                FALSE
    │                   │
    ▼                   ▼
┌───────────────┐  ┌─────────────────┐
│ Data Table /  │  │  Send Email     │
│    Insert     │  │     (Gmail)     │
└───────┬───────┘  └────────┬────────┘
        │                   │
        └─────────┬─────────┘
                  │
                  ▼
        ┌──────────────────────┐
        │ Respond to Webhook   │
        └──────────────────────┘
```

### Flujo resumido

```text
[ Webhook POST ]
       ↓
[ Code (JS) ]
       ↓
[ IF / Switch ]
    ↙       ↘
 TRUE      FALSE
  ↓          ↓
[Data]    [Gmail]
    ↘      ↙
 [Respond to Webhook]
```

---

# 🛠️ Configuración Paso a Paso

## 1. Nodo Webhook

Agregar un nodo **Webhook** como punto de entrada del flujo.

### Configuración

| Propiedad | Valor |
|---|---|
| **HTTP Method** | `POST` |
| **Path** | `evaluar-solicitud` |
| **Respond** | `Using 'Respond to Webhook' Node` |

El Webhook recibirá una solicitud HTTP con la información de la solicitud de crédito.

### Ejemplo de datos recibidos

```json
{
  "solicitante": "Aura Alba",
  "monto": 15000,
  "score_crediticio": 750
}
```

---

# 2. Nodo Code (JavaScript)

Agregar un nodo **Code** después del Webhook.

Cambiar el selector de lenguaje a:

```text
JavaScript
```

El nodo será responsable de procesar la información recibida y ejecutar las reglas de negocio.

## Código

```javascript
const items = $input.all();
const resultados = [];

for (const item of items) {

  const data = item.json.body || {};

  const monto = data.monto || 0;
  const score = data.score_crediticio || 0;
  const solicitante = data.solicitante || 'Desconocido';

  let estado = '';
  let motivo = '';

  if (monto <= 0) {

    estado = "RECHAZADO_ERROR_DATOS";
    motivo = "El monto debe ser superior a 0";

  } else if (monto > 50000 && score < 700) {

    estado = "RECHAZADO_ALTO_RIESGO";
    motivo = "Monto elevado sin score crediticio suficiente";

  } else {

    estado = "APROBADO";
    motivo = "Cumple criterios automáticos";
  }

  resultados.push({
    json: {
      solicitante,
      monto,
      score,
      estado,
      motivo
    }
  });
}

return resultados;
```

---

# 🧠 Reglas de Negocio

El nodo `Code` aplica las siguientes reglas:

### Regla 1 — Datos inválidos

Si:

```text
monto <= 0
```

El resultado será:

```text
Estado: RECHAZADO_ERROR_DATOS
```

Motivo:

```text
El monto debe ser superior a 0
```

---

### Regla 2 — Alto riesgo

Si:

```text
monto > 50000
AND
score < 700
```

El resultado será:

```text
Estado: RECHAZADO_ALTO_RIESGO
```

Motivo:

```text
Monto elevado sin score crediticio suficiente
```

---

### Regla 3 — Aprobación automática

Si ninguna de las reglas anteriores se cumple:

```text
Estado: APROBADO
```

Motivo:

```text
Cumple criterios automáticos
```

---

# 🔀 3. Nodo IF / Switch

Después del nodo `Code`, agregar un nodo **IF** o **Switch** para enrutar el flujo dependiendo del resultado obtenido.

El campo principal para evaluar es:

```text
estado
```

### Condición de ejemplo

```text
estado == "APROBADO"
```

El flujo puede dividirse de la siguiente manera:

```text
                    ┌─── APROBADO ───→ Data Table / Insert
                    │
[ Code ] → [ IF ] ──┤
                    │
                    └─── RECHAZADO ──→ Send Email (Gmail)
```

---

# 🗄️ 4. Data Table / Insert

Para las solicitudes aprobadas, conectar la salida **TRUE** del nodo `IF` con el nodo encargado de almacenar el registro.

Por ejemplo:

```text
[ IF - TRUE ]
      ↓
[ Data Table / Insert ]
```

Los datos que pueden almacenarse son:

| Campo | Valor |
|---|---|
| `solicitante` | Nombre del solicitante |
| `monto` | Monto solicitado |
| `score` | Score crediticio |
| `estado` | Resultado de evaluación |
| `motivo` | Justificación del resultado |

---

# 📧 5. Send Email (Gmail)

Para solicitudes que no cumplen los criterios de aprobación, conectar la salida **FALSE** con un nodo de envío de correo.

```text
[ IF - FALSE ]
      ↓
[ Send Email (Gmail) ]
```

El correo puede utilizar la información generada por el nodo `Code`, por ejemplo:

```text
Solicitante: {{$json.solicitante}}

Monto solicitado: {{$json.monto}}

Score crediticio: {{$json.score}}

Estado: {{$json.estado}}

Motivo: {{$json.motivo}}
```

Esto permite notificar automáticamente cuando una solicitud debe ser rechazada o requiere atención.

---

# 🔄 6. Respond to Webhook

Finalmente, conectar el flujo con:

```text
Respond to Webhook
```

Este nodo devuelve la respuesta al sistema que realizó la solicitud HTTP.

Un ejemplo de respuesta podría ser:

```json
{
  "solicitante": "Aura Alba",
  "monto": 15000,
  "score": 750,
  "estado": "APROBADO",
  "motivo": "Cumple criterios automáticos"
}
```

---

# 🧪 Probando el Flujo vía cURL

## 1. Ejecutar el Workflow

En n8n:

1. Abrir el workflow.
2. Hacer clic en **Execute workflow**.
3. El Webhook quedará preparado para recibir la solicitud de prueba.

---

## 2. Obtener la URL del Webhook

Copiar la URL de prueba generada por el nodo **Webhook**.

Ejemplo:

```text
<TU_TEST_WEBHOOK_URL>
```

---

## 3. Ejecutar la solicitud desde la terminal

Utilizar el siguiente comando:

```bash
curl -X POST "<TU_TEST_WEBHOOK_URL>" \
  -H "Content-Type: application/json" \
  -d '{
    "solicitante": "Aura Alba",
    "monto": 15000,
    "score_crediticio": 750
  }'
```

> Reemplazar `<TU_TEST_WEBHOOK_URL>` por la URL proporcionada por el nodo Webhook de n8n.

---

# 📊 Ejemplos de Prueba

## ✅ Caso 1 — Solicitud aprobada

### Request

```json
{
  "solicitante": "Aura Alba",
  "monto": 15000,
  "score_crediticio": 750
}
```

### Resultado esperado

```json
{
  "solicitante": "Aura Alba",
  "monto": 15000,
  "score": 750,
  "estado": "APROBADO",
  "motivo": "Cumple criterios automáticos"
}
```

Flujo:

```text
Webhook
   ↓
Code
   ↓
IF → TRUE
   ↓
Data Table / Insert
   ↓
Respond to Webhook
```

---

## ❌ Caso 2 — Monto inválido

### Request

```json
{
  "solicitante": "Aura Alba",
  "monto": 0,
  "score_crediticio": 750
}
```

### Resultado esperado

```json
{
  "solicitante": "Aura Alba",
  "monto": 0,
  "score": 750,
  "estado": "RECHAZADO_ERROR_DATOS",
  "motivo": "El monto debe ser superior a 0"
}
```

---

## ⚠️ Caso 3 — Alto riesgo

### Request

```json
{
  "solicitante": "Aura Alba",
  "monto": 75000,
  "score_crediticio": 650
}
```

### Resultado esperado

```json
{
  "solicitante": "Aura Alba",
  "monto": 75000,
  "score": 650,
  "estado": "RECHAZADO_ALTO_RIESGO",
  "motivo": "Monto elevado sin score crediticio suficiente"
}
```

Flujo:

```text
Webhook
   ↓
Code
   ↓
IF → FALSE
   ↓
Send Email (Gmail)
   ↓
Respond to Webhook
```

---

# 🔗 Flujo Completo

La automatización completa puede representarse de la siguiente manera:

```text
                    ┌─────────────────────┐
                    │    Webhook POST     │
                    │ evaluar-solicitud   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │      Code (JS)      │
                    │                     │
                    │  Evalúa las reglas  │
                    │    de negocio       │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │     IF / Switch     │
                    │                     │
                    │ estado = APROBADO?  │
                    └──────┬───────┬──────┘
                           │       │
                    TRUE   │       │   FALSE
                           │       │
                           ▼       ▼
                 ┌─────────────┐ ┌─────────────┐
                 │ Data Table  │ │ Send Email  │
                 │   Insert    │ │   (Gmail)   │
                 └──────┬──────┘ └──────┬──────┘
                        │               │
                        └───────┬───────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │ Respond to Webhook  │
                    └─────────────────────┘
```

---

# 🧩 Componentes del Workflow

| Nodo | Responsabilidad |
|---|---|
| **Webhook** | Recibir solicitudes HTTP |
| **Code** | Ejecutar las reglas de negocio |
| **IF / Switch** | Enrutar según el resultado |
| **Data Table / Insert** | Almacenar solicitudes aprobadas |
| **Send Email (Gmail)** | Notificar resultados no aprobados |
| **Respond to Webhook** | Devolver la respuesta al cliente |

---

# 📌 Resumen

Este ejercicio demuestra cómo utilizar **n8n para implementar una arquitectura orientada a eventos**, donde una solicitud externa puede iniciar automáticamente un flujo de procesamiento.

La solución integra:

- 🌐 **Webhooks** para recibir eventos.
- ⚙️ **JavaScript** para implementar reglas de negocio.
- 🔀 **IF / Switch** para tomar decisiones.
- 🗄️ **Persistencia** mediante Data Table.
- 📧 **Notificaciones automáticas** mediante Gmail.
- 🔄 **Respond to Webhook** para devolver el resultado.

El flujo permite separar la recepción del evento, el procesamiento de las reglas y las acciones posteriores, facilitando la construcción de procesos automatizados y extensibles.