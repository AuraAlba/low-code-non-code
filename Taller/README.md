# 📝 Taller Práctico: Orquestación de Incidentes Críticos y Despliegues (LCNC)

## 📌 Caso de Uso de Negocio

Un sistema de monitoreo de infraestructura, como **Datadog**, **Prometheus** o un **Healthcheck de GitHub Actions**, envía una alerta en formato **JSON** cuando ocurre un fallo en un entorno de producción o staging.

El objetivo del taller es construir un flujo de automatización utilizando una herramienta **Low-Code/No-Code (LCNC)**, seleccionando entre **n8n** o **Appsmith**, que permita:

1. Recibir una alerta de infraestructura.
2. Procesar el payload recibido.
3. Clasificar automáticamente la gravedad del incidente.
4. Ejecutar acciones según el nivel de criticidad.
5. Registrar los eventos generados.
6. Notificar los incidentes críticos.

---

# 🎯 Guía Paso a Paso

## 1. Selección de la Herramienta

Seleccionar una de las siguientes plataformas:

### Opción A — n8n

Plataforma enfocada principalmente en **automatización de procesos y eventos**.

Para este ejercicio se utilizará:

- Nodo **Webhook** para recibir las alertas.
- Nodo **Code** para procesar la lógica de negocio.
- Nodo **IF / Switch** para enrutar los eventos.
- Nodo de almacenamiento para registrar los incidentes.
- Nodo de notificación para alertas críticas.

Flujo general:

```text
[ Webhook ]
     ↓
[ Code - JavaScript ]
     ↓
[ IF / Switch ]
   ↙       ↘
CRÍTICO   ADVERTENCIA / INFO
  ↓              ↓
[Registro]    [Historial]
  ↓
[Notificación]
```

---

### Opción B — Appsmith

Plataforma enfocada en la construcción de **interfaces y aplicaciones internas**.

Para este ejercicio se puede construir:

- Un formulario simulador de alertas.
- Un JS Object para procesar la lógica.
- Un panel de incidentes.
- Un historial de eventos.
- Indicadores visuales según la criticidad.
- Notificaciones para incidentes críticos.

Flujo general:

```text
[ Formulario de Alerta ]
          ↓
[ JS Object ]
          ↓
[ Clasificación ]
     ↙         ↘
 CRÍTICO    ADVERTENCIA / INFO
    ↓             ↓
[Registro]    [Historial]
    ↓
[Notificación]
```

---

# 📥 2. Captura y Procesamiento del Payload

El flujo debe estar preparado para recibir un objeto **JSON** con la siguiente estructura:

```json
{
  "servicio": "auth-service",
  "entorno": "produccion",
  "codigo_error": 500,
  "latencia_ms": 1250,
  "desarrollador_responsable": "Tu Nombre"
}
```

## 📋 Descripción de los campos

| Campo | Tipo | Descripción |
|---|---|---|
| `servicio` | String | Nombre del servicio afectado |
| `entorno` | String | Entorno donde ocurrió el incidente |
| `codigo_error` | Number | Código HTTP o código de error |
| `latencia_ms` | Number | Latencia registrada en milisegundos |
| `desarrollador_responsable` | String | Responsable del servicio o incidente |

---

# 🧠 3. Lógica de Negocio

La solución debe implementar una función en **JavaScript** que analice los datos recibidos y determine el nivel de criticidad del incidente.

## 🔴 CRÍTICO

Un incidente debe clasificarse como **CRÍTICO** cuando:

```text
entorno == "produccion"
AND
(
    codigo_error >= 500
    OR
    latencia_ms > 1000
)
```

### Ejemplo

```json
{
  "servicio": "auth-service",
  "entorno": "produccion",
  "codigo_error": 500,
  "latencia_ms": 1250,
  "desarrollador_responsable": "Tu Nombre"
}
```

Resultado:

```text
CRÍTICO
```

---

## 🟡 ADVERTENCIA

Un incidente debe clasificarse como **ADVERTENCIA** cuando:

```text
entorno == "staging"
OR
latencia_ms >= 500 AND latencia_ms <= 1000
```

### Ejemplo

```json
{
  "servicio": "payment-service",
  "entorno": "staging",
  "codigo_error": 200,
  "latencia_ms": 700,
  "desarrollador_responsable": "Tu Nombre"
}
```

Resultado:

```text
ADVERTENCIA
```

---

## 🔵 INFO

Cualquier escenario que no cumpla las condiciones anteriores debe clasificarse como:

```text
INFO
```

### Ejemplo

```json
{
  "servicio": "catalog-service",
  "entorno": "produccion",
  "codigo_error": 200,
  "latencia_ms": 200,
  "desarrollador_responsable": "Tu Nombre"
}
```

Resultado:

```text
INFO
```

---

# 💻 Implementación de la Lógica en JavaScript

La siguiente función puede utilizarse como base para implementar la clasificación:

```javascript
function clasificarIncidente(payload) {

  const {
    servicio,
    entorno,
    codigo_error,
    latencia_ms,
    desarrollador_responsable
  } = payload;

  let criticidad = "";
  let motivo = "";

  if (
    entorno === "produccion" &&
    (codigo_error >= 500 || latencia_ms > 1000)
  ) {

    criticidad = "CRÍTICO";
    motivo = "Error crítico o latencia elevada en producción.";

  } else if (
    entorno === "staging" ||
    (latencia_ms >= 500 && latencia_ms <= 1000)
  ) {

    criticidad = "ADVERTENCIA";
    motivo = "El incidente requiere seguimiento.";

  } else {

    criticidad = "INFO";
    motivo = "Evento informativo sin condiciones críticas.";
  }

  return {
    servicio,
    entorno,
    codigo_error,
    latencia_ms,
    desarrollador_responsable,
    criticidad,
    motivo
  };
}
```

---

# 🔀 4. Enrutamiento y Acciones

Después de clasificar el incidente, el flujo debe tomar una decisión según el nivel de criticidad.

## 🔴 Si el resultado es CRÍTICO

Cuando:

```text
criticidad == "CRÍTICO"
```

se deben ejecutar las siguientes acciones:

1. Registrar el incidente en una tabla o base de datos.
2. Generar una notificación.
3. Enviar una alerta al responsable.

Ejemplo de flujo:

```text
[Clasificación]
      ↓
  CRÍTICO
      ↓
[Registrar Incidente]
      ↓
[Enviar Notificación]
```

La notificación puede ser:

- Gmail.
- Slack.
- Microsoft Teams.
- Una alerta en pantalla.
- Otro mecanismo de notificación disponible.

---

# 🟡🔵 Si el resultado es ADVERTENCIA o INFO

Cuando el resultado sea:

```text
ADVERTENCIA
```

o:

```text
INFO
```

se debe guardar únicamente el evento en el historial o log general.

```text
[Clasificación]
      ↓
ADVERTENCIA / INFO
      ↓
[Historial / Log]
```

No es necesario generar una notificación crítica.

---

# 🔄 Flujo Completo

La arquitectura general del ejercicio puede representarse de la siguiente manera:

```text
                         ┌────────────────────┐
                         │ Sistema Monitoreo  │
                         │ Datadog / Prometheus│
                         │ GitHub Actions     │
                         └─────────┬──────────┘
                                   │
                                   │ JSON
                                   ▼
                         ┌────────────────────┐
                         │ Webhook / Formulario│
                         └─────────┬──────────┘
                                   │
                                   ▼
                         ┌────────────────────┐
                         │  Code / JS Object  │
                         │                    │
                         │ Clasificar evento  │
                         └─────────┬──────────┘
                                   │
                                   ▼
                         ┌────────────────────┐
                         │  IF / Switch       │
                         └─────────┬──────────┘
                                   │
                 ┌─────────────────┼─────────────────┐
                 │                 │                 │
                 ▼                 ▼                 ▼
             CRÍTICO          ADVERTENCIA          INFO
                 │                 │                 │
                 ▼                 └────────┬────────┘
        ┌─────────────────┐                 │
        │ Registrar       │                 ▼
        │ incidente       │          ┌──────────────┐
        └────────┬────────┘          │ Historial /  │
                 │                   │     Log      │
                 ▼                   └──────────────┘
        ┌─────────────────┐
        │ Notificación    │
        │     crítica     │
        └─────────────────┘
```

---

# 🧪 5. Validación y Pruebas

La solución debe probarse utilizando al menos **dos escenarios diferentes**:

- Un payload que genere una alerta **CRÍTICA**.
- Un payload que genere una alerta **ADVERTENCIA** o **INFO**.

Las pruebas pueden realizarse mediante:

- **cURL**
- **Postman**
- Formulario de prueba de Appsmith.

---

## 🔴 Escenario 1 — CRÍTICO

### Payload

```json
{
  "servicio": "auth-service",
  "entorno": "produccion",
  "codigo_error": 500,
  "latencia_ms": 1250,
  "desarrollador_responsable": "Tu Nombre"
}
```

### Resultado esperado

```text
Criticidad: CRÍTICO
```

### Acciones esperadas

```text
✓ Registrar incidente
✓ Generar notificación
✓ Asociar responsable
```

---

## 🟡 Escenario 2 — ADVERTENCIA

### Payload

```json
{
  "servicio": "payment-service",
  "entorno": "staging",
  "codigo_error": 200,
  "latencia_ms": 700,
  "desarrollador_responsable": "Tu Nombre"
}
```

### Resultado esperado

```text
Criticidad: ADVERTENCIA
```

### Acciones esperadas

```text
✓ Registrar evento en historial
✗ No generar alerta crítica
```

---

## 🔵 Escenario 3 — INFO

### Payload

```json
{
  "servicio": "catalog-service",
  "entorno": "produccion",
  "codigo_error": 200,
  "latencia_ms": 200,
  "desarrollador_responsable": "Tu Nombre"
}
```

### Resultado esperado

```text
Criticidad: INFO
```

### Acciones esperadas

```text
✓ Registrar evento en historial
✗ No generar alerta crítica
```

---

# 🧪 Prueba mediante cURL

Si se utiliza **n8n**, se puede probar el Webhook utilizando el siguiente comando:

```bash
curl -X POST "<TU_TEST_WEBHOOK_URL>" \
  -H "Content-Type: application/json" \
  -d '{
    "servicio": "auth-service",
    "entorno": "produccion",
    "codigo_error": 500,
    "latencia_ms": 1250,
    "desarrollador_responsable": "Tu Nombre"
  }'
```

> Reemplazar `<TU_TEST_WEBHOOK_URL>` por la URL correspondiente al Webhook configurado en n8n.

---

# 📦 Entregables del Taller

Se debe entregar **un único documento en formato PDF o Word** que contenga las evidencias de la implementación y las respuestas al cuestionario de reflexión.

---

## 1. 📸 Evidencias Gráficas

El documento debe incluir las siguientes capturas:

### 1.1 Flujo o interfaz

Captura de pantalla que muestre la solución construida en la herramienta seleccionada.

**n8n:**

- Vista general del workflow.
- Nodos configurados.
- Conexiones entre los nodos.

**Appsmith:**

- Vista general de la interfaz.
- Formulario de alertas.
- Panel de incidentes.

---

### 1.2 Lógica JavaScript

Captura de pantalla donde se evidencie la implementación de la lógica utilizada para clasificar la criticidad.

Debe ser posible identificar las condiciones:

```text
CRÍTICO
ADVERTENCIA
INFO
```

---

### 1.3 Prueba del escenario CRÍTICO

La evidencia debe mostrar:

- Payload enviado.
- Resultado de la clasificación.
- Registro generado.
- Notificación generada.

---

### 1.4 Prueba del escenario ADVERTENCIA / INFO

La evidencia debe mostrar:

- Payload enviado.
- Resultado de la clasificación.
- Registro en el historial o log.

---

### 1.5 Comando o herramienta utilizada para las pruebas

Incluir evidencia de:

- Comando `cURL`, o
- Colección/request de **Postman**, o
- Formulario utilizado en **Appsmith**.

---

# 📝 2. Cuestionario de Reflexión y Experiencia

Al final del documento se deben responder las siguientes preguntas.

---

## Pregunta 1

**¿Cuáles fueron las principales ventajas y limitaciones que encontraste al implementar la lógica de negocio utilizando una herramienta Low-Code/No-Code en comparación con el desarrollo tradicional (código desde cero)?**

### Aspectos que se pueden analizar

- Velocidad de implementación.
- Facilidad de configuración.
- Curva de aprendizaje.
- Reutilización de componentes.
- Integración con otros servicios.
- Control sobre la lógica.
- Flexibilidad.
- Mantenimiento.
- Dependencia de la plataforma.
- Escalabilidad.

---

## Pregunta 2

**Según tu experiencia en este ejercicio, ¿en qué casos de uso reales de un entorno de TI recomendarías el uso de n8n y en cuáles el de Appsmith?**

### Aspectos que se pueden analizar

**n8n:**

- Automatización de procesos.
- Integración entre sistemas.
- Procesamiento de eventos.
- Webhooks.
- Notificaciones.
- Flujos de integración.
- Procesos programados.

**Appsmith:**

- Aplicaciones internas.
- Dashboards.
- Paneles administrativos.
- Formularios.
- Herramientas de operación.
- Interfaces para equipos de negocio.

---

## Pregunta 3

**¿Qué impacto consideras que tienen estas plataformas en la velocidad de entrega (Time-to-Market) y en la colaboración entre equipos técnicos y de negocio?**

### Aspectos que se pueden analizar

- Reducción del tiempo de desarrollo.
- Prototipado rápido.
- Participación de usuarios de negocio.
- Reducción de tareas repetitivas.
- Integración entre equipos.
- Automatización de procesos.
- Dependencia de conocimientos técnicos.
- Evolución y mantenimiento de las soluciones.

---

# 📋 Checklist de Entrega

Antes de entregar el documento, verificar que incluya:

- [ ] Herramienta seleccionada: **n8n o Appsmith**.
- [ ] Payload JSON configurado.
- [ ] Lógica de clasificación implementada.
- [ ] Clasificación **CRÍTICO** probada.
- [ ] Clasificación **ADVERTENCIA o INFO** probada.
- [ ] Registro de incidentes implementado.
- [ ] Notificación para incidentes críticos implementada.
- [ ] Evidencia del flujo o interfaz.
- [ ] Evidencia del código JavaScript.
- [ ] Evidencia del escenario CRÍTICO.
- [ ] Evidencia del escenario ADVERTENCIA / INFO.
- [ ] Evidencia de cURL, Postman o formulario.
- [ ] Respuesta a las tres preguntas de reflexión.
- [ ] Documento final en **PDF o Word**.