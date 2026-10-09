# ⚡ Demostración de Automatización de Eventos con n8n

## 🎯 Objetivo

Configurar un flujo de trabajo en **n8n** que permita:

- Recibir una solicitud mediante un **Webhook HTTP POST**.
- Procesar los datos recibidos.
- Ejecutar reglas de negocio mediante código python.
- Determinar automáticamente el resultado de una solicitud de crédito.
- Enrutar el flujo según el resultado de la evaluación.
- Almacenar las solicitudes aprobadas en una Data Table.
- Enviar una notificación por correo electrónico cuando una solicitud sea rechazada.
- Responder al sistema que realizó la solicitud.

## 📐 Arquitectura del flujo

```text
                 ┌──────────────────────┐
                 │     Webhook POST     │
                 │  evaluar-solicitud   │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │        Code          │
                 │  Reglas de negocio   │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │         IF           │
                 │ ¿Estado = APROBADO?  │
                 └────────┬───────┬─────┘
                          │       │
                    TRUE  │       │ FALSE
                          ▼       ▼
                 ┌────────────┐ ┌──────────────┐
                 │ Data Table │ │ Gmail: enviar│
                 │ Insert Row │ │ notificación │
                 └──────┬─────┘ └──────┬───────┘
                        │              │
                        └──────┬───────┘
                               ▼
                 ┌──────────────────────┐
                 │  Respond to Webhook  │
                 └──────────────────────┘
```

## 🛠️ Configuración paso a paso

### 1. Crear una cuenta en n8n

1. Ingresa a [https://app.n8n.cloud/register](https://app.n8n.cloud/register).
2. Regístrate y completa el proceso de acceso.
3. Una vez dentro de la plataforma, selecciona la opción para crear un nuevo workflow.
4. Asigna un nombre al flujo, por ejemplo: `Evaluacion de solicitudes de credito`.

### 2. Configurar el Webhook de entrada

El Webhook será el punto de entrada de las solicitudes. Permitirá que aplicaciones externas envíen información a n8n mediante una petición HTTP.

**Paso 1. Agregar el nodo**

1. En el editor del workflow, haz clic en `+`.
2. Busca el nodo **Webhook** y agrégalo al flujo.

**Paso 2. Configurar los parámetros**

| Propiedad | Valor |
|---|---|
| HTTP Method | `POST` |
| Path | `evaluar-solicitud` |
| Respond | `Using 'Respond to Webhook' Node` |

Con esta configuración, el flujo recibirá solicitudes HTTP POST y delegará la respuesta final al nodo `Respond to Webhook`.

**Paso 3. Identificar la URL**

Abre el nodo Webhook y localiza las URL disponibles:

- **Test URL:** se utiliza durante las pruebas manuales.
- **Production URL:** se utiliza cuando el workflow está activo.

Copia la URL que corresponda al tipo de ejecución que estés realizando.

**Ejemplo de datos que recibirá el Webhook:**

```json
{
  "solicitante": "Aura Alba",
  "monto": 15000,
  "score_crediticio": 750,
  "email": "solicitante@example.com"
}
```

El correo electrónico es opcional para la evaluación, pero será necesario si deseas enviar la notificación al solicitante de forma dinámica.

### 3. Implementar la lógica de negocio con Code

El nodo `Code` ejecutará las reglas que determinan si una solicitud de crédito puede aprobarse automáticamente o debe rechazarse.

**Paso 1. Agregar el nodo**

1. Haz clic en el `+` situado después del Webhook.
2. Busca y agrega el nodo **Code**.
3. Conecta la salida del Webhook con la entrada del nodo Code.

**Paso 2. Configurar el lenguaje**

En las propiedades del nodo, selecciona:

- **Language:** `Python`.
- **Mode:** `Run Once for All Items`.

**Paso 3. Ingresar el código**

Copia el siguiente código en el editor:

```python
resultados = []

for item in _items:
    # Obtener los datos enviados por el Webhook
    raw_json = item.get('json', {})
    data = raw_json.get('body', raw_json)

    monto = data.get('monto', 0)
    score = data.get('score_crediticio', 0)
    solicitante = data.get('solicitante', 'Desconocido')
    email = data.get('email', '')

    # Convertir los valores numéricos
    try:
        monto = float(monto)
        score = float(score)
    except (ValueError, TypeError):
        monto = 0
        score = -1

    # Aplicar las reglas de negocio
    if monto <= 0:
        estado = "RECHAZADO_ERROR_DATOS"
        motivo = "El monto debe ser superior a 0"

    elif score < 0 or score > 1000:
        estado = "RECHAZADO_ERROR_DATOS"
        motivo = "El score crediticio debe estar entre 0 y 1000"

    elif monto > 50000 and score < 700:
        estado = "RECHAZADO_ALTO_RIESGO"
        motivo = "Monto elevado sin score crediticio suficiente"

    else:
        estado = "APROBADO"
        motivo = "Cumple criterios automáticos"

    # Construir la salida del nodo
    resultados.append({
        "json": {
            "solicitante": solicitante,
            "email": email,
            "monto": monto,
            "score": score,
            "estado": estado,
            "motivo": motivo
        }
    })

return resultados
```

**¿Qué hace el código?**

1. Lee los datos que recibe del Webhook.
2. Extrae el solicitante, el monto, el score crediticio y el correo electrónico.
3. Evalúa las reglas de negocio en orden.
4. Construye un resultado con el estado y la justificación.
5. Devuelve la información para que los siguientes nodos puedan utilizarla.

### 4. Configurar el enrutamiento condicional con IF

El nodo `IF` permitirá separar las solicitudes aprobadas de las rechazadas.

**Paso 1. Agregar el nodo**

1. Haz clic en `+` después del nodo Code.
2. Busca y agrega **IF**.
3. Conecta la salida del nodo Code con la entrada del nodo IF.

**Paso 2. Definir la condición**

Configura los siguientes valores:

| Campo | Valor |
|---|---|
| Value 1 | `{{ $json.estado }}` |
| Operation | `String` → `Equal` |
| Value 2 | `APROBADO` |

La interfaz puede presentar la operación con una etiqueta ligeramente diferente según la versión de n8n.

**Paso 3. Interpretar las salidas**

- **True:** la solicitud cumple los criterios automáticos de aprobación.
- **False:** la solicitud fue rechazada por datos inválidos o por alto riesgo.

Las dos salidas permitirán ejecutar acciones diferentes según el resultado de la evaluación.

### 5. Rama TRUE: almacenar las solicitudes aprobadas

Las solicitudes aprobadas se guardarán en una Data Table de n8n.

#### Paso 1. Crear la Data Table

1. Regresa al menú principal de n8n.
2. Busca la sección **Data tables** o la opción **New data table**, según la versión de la plataforma.
3. Crea una tabla con el nombre:

   `solicitudes_credito`

4. Configura las siguientes columnas:

| Columna | Tipo |
|---|---|
| `solicitante` | String |
| `monto` | Number |
| `score` | Number |
| `estado` | String |
| `motivo` | String |

5. Guarda o crea la tabla.

#### Paso 2. Agregar el nodo Data Table

1. Regresa al workflow.
2. En la salida **True** del nodo IF, haz clic en `+`.
3. Busca y agrega el nodo **Data Table**.
4. Selecciona la tabla `solicitudes_credito`.
5. En la operación, selecciona **Insert Row** o la opción equivalente para insertar una fila.

#### Paso 3. Mapear los datos

En los campos de la operación, asigna las expresiones correspondientes:

| Columna de destino | Expresión |
|---|---|
| `solicitante` | `{{ $json.solicitante }}` |
| `monto` | `{{ $json.monto }}` |
| `score` | `{{ $json.score }}` |
| `estado` | `{{ $json.estado }}` |
| `motivo` | `{{ $json.motivo }}` |

Utiliza el modo de expresión de cada campo para introducir los valores dinámicos.

**Resultado esperado:** cuando una solicitud sea aprobada, se insertará un registro en la tabla.

### 6. Rama FALSE: enviar un correo mediante Gmail

Las solicitudes rechazadas activarán una notificación por correo electrónico.

#### Paso 1. Agregar el nodo Gmail

1. En la salida **False** del nodo IF, haz clic en `+`.
2. Busca el nodo **Gmail**.
3. Selecciona la operación para enviar un mensaje, normalmente **Send a message** o **Send**.

#### Paso 2. Conectar la cuenta de Google

1. En el apartado de credenciales, selecciona **Create new credential** o la opción equivalente.
2. Sigue el proceso de autenticación y autorización de Google.
3. Guarda las credenciales y verifica que el nodo pueda utilizar la cuenta.

La conexión puede requerir una configuración adicional de Google OAuth, según el tipo de cuenta y las opciones disponibles en tu instancia de n8n.

#### Paso 3. Configurar el destinatario

En el campo **To**, puedes usar un correo fijo para la demostración.

Para enviar el mensaje al correo incluido en el JSON de entrada, utiliza esta expresión:

```javascript
{{ $json.email }}
```

Si el correo no está incluido en la solicitud, asegúrate de configurar un destinatario de prueba válido antes de ejecutar la rama de rechazo.

#### Paso 4. Configurar el asunto

En **Subject**, escribe:

```text
Actualización de tu solicitud de crédito - Rechazada
```

En **Message Type**, selecciona `Text` o la opción equivalente para texto plano.

#### Paso 5. Configurar el cuerpo del correo

En el campo **Message** o **Body**, utiliza el siguiente contenido:

```text
                Hola {{ $json.solicitante }},                      Lamentamos informarte que tu solicitud de crédito por un monto de ${{ $json.monto }} ha sido rechazada.                      Motivo del sistema: {{ $json.motivo }}                      Si tienes dudas, por favor contáctanos.
```

Las expresiones consultan directamente el resultado del nodo Code, de modo que los datos originales sigan disponibles aunque el nodo Gmail genere una respuesta propia.

**Resultado esperado:** cada solicitud que siga la rama False enviará un correo con el nombre, el monto y el motivo del rechazo.

### 7. Configurar Respond to Webhook

Este nodo devolverá el resultado de la evaluación al sistema que envió la solicitud HTTP.

#### Paso 1. Agregar el nodo

1. Busca el nodo **Respond to Webhook**.
2. Conecta a él las salidas finales de las dos ramas:
   - La salida del nodo Data Table.
   - La salida del nodo Gmail.

Asegúrate de que ambas ramas lleguen al nodo de respuesta. Solo se ejecutará la rama que corresponda a la condición evaluada.

#### Paso 2. Configurar la respuesta

En el nodo Respond to Webhook, configura:

- **Respond With:** `JSON`.
- **Response Code:** activa el modo de expresión y utiliza:

```javascript
{{  $json.estado === 'APROBADO' ? 200 : 400}}
```

Esto devuelve HTTP `200` cuando la solicitud es aprobada y HTTP `400` cuando se rechaza.

> Nota: HTTP 400 representa un rechazo de la solicitud en esta demostración, pero en una API de producción normalmente se reserva para solicitudes incorrectas. Un rechazo de negocio puede comunicarse también con HTTP 200 y un campo `estado` que indique el resultado.

#### Paso 3. Definir el cuerpo de la respuesta

En el campo **Response Body**, utiliza el modo de expresión para construir el siguiente objeto JSON:

```javascript
={{
  {
     "status": "{{ $json.estado }}",
      "applicant": "{{ $json.solicitante }}",
      "amount": {{ $json.monto }},
      "message": "{{ $json.motivo }}",
      "timestamp": "{{ $now.toISOString() }}"
  }
}}
```

Esta configuración consulta el resultado del nodo Code, independientemente de que la rama haya pasado por Data Table o por Gmail.

**Ejemplo de respuesta para una solicitud aprobada:**

```json
{
  "status": "APROBADO",
  "applicant": "Aura Alba",
  "amount": 15000,
  "score": 750,
  "message": "Cumple criterios automáticos",
  "timestamp": "2026-10-09T12:00:00.000Z"
}
```

La fecha y hora del ejemplo son ilustrativas; n8n generará el valor real durante la ejecución.

## 🧪 Probar el workflow con cURL

### 1. Probar el Webhook en modo de prueba

1. Abre el nodo Webhook.
2. Haz clic en **Listen for test event** o en la opción equivalente para escuchar una solicitud de prueba.
3. Copia la **Test URL** del nodo.

La URL de prueba suele incluir `/webhook-test/`. Utiliza la dirección completa que muestre tu instancia.

### 2. Enviar una solicitud desde la terminal

Reemplaza `TU_TEST_WEBHOOK_URL` por la URL copiada del nodo Webhook.

```bash
curl -X POST "TU_TEST_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{
    "solicitante": "Pepita Perez",
    "monto": 10000,
    "score_crediticio": 800,
    "email": "correo-de-prueba@example.com"
  }'
```

Puedes ejecutar el comando desde una terminal Linux, macOS o desde una herramienta como Postman.

**Importante:** el Webhook de prueba debe estar escuchando cuando envíes la petición. Para utilizar la URL de producción, activa el workflow y usa la Production URL correspondiente.

## 📊 Casos de prueba

### Caso 1: solicitud aprobada

**Entrada:**

```json
{
  "solicitante": "Aura Alba",
  "monto": 15000,
  "score_crediticio": 750,
  "email": "correo-de-prueba@example.com"
}
```

**Resultado esperado:**

```json
{
  "status": "APROBADO",
  "applicant": "Aura Alba",
  "amount": 15000,
  "score": 750,
  "message": "Cumple criterios automáticos"
}
```

**Recorrido:** Webhook → Code → IF (True) → Data Table → Respond to Webhook.

### Caso 2: monto inválido

**Entrada:**

```json
{
  "solicitante": "Aura Alba",
  "monto": 0,
  "score_crediticio": 750,
  "email": "correo-de-prueba@example.com"
}
```

**Resultado esperado:**

```json
{
  "status": "RECHAZADO_ERROR_DATOS",
  "applicant": "Aura Alba",
  "amount": 0,
  "score": 750,
  "message": "El monto debe ser superior a 0"
}
```

**Recorrido:** Webhook → Code → IF (False) → Gmail → Respond to Webhook.

### Caso 3: solicitud de alto riesgo

**Entrada:**

```json
{
  "solicitante": "Pepita Perez",
  "monto": 75000,
  "score_crediticio": 650,
  "email": "correo-de-prueba@example.com"
}
```

**Resultado esperado:**

```json
{
  "status": "RECHAZADO_ALTO_RIESGO",
  "applicant": "Pepita Perez",
  "amount": 75000,
  "score": 650,
  "message": "Monto elevado sin score crediticio suficiente"
}
```

**Recorrido:** Webhook → Code → IF (False) → Gmail → Respond to Webhook.

### Caso 4: monto elevado con score suficiente

**Entrada:**

```json
{
  "solicitante": "Carlos Perez",
  "monto": 75000,
  "score_crediticio": 750,
  "email": "correo-de-prueba@example.com"
}
```

**Resultado esperado:** `APROBADO`, porque la regla de rechazo por alto riesgo requiere que el monto sea superior a 50.000 y el score sea inferior a 700.

**Recorrido:** Webhook → Code → IF (True) → Data Table → Respond to Webhook.

## 🧩 Componentes del workflow

| Nodo | Responsabilidad |
|---|---|
| Webhook | Recibir solicitudes HTTP POST |
| Code | Ejecutar las reglas de negocio |
| IF | Separar solicitudes aprobadas y rechazadas |
| Data Table | Almacenar solicitudes aprobadas |
| Gmail | Enviar notificaciones de rechazo |
| Respond to Webhook | Devolver el resultado al cliente |

## 📌 Conclusión

Este ejercicio demuestra cómo utilizar **n8n para construir una automatización orientada a eventos**, en la que una solicitud externa inicia un flujo de procesamiento que ejecuta reglas de negocio, toma decisiones, almacena datos y activa notificaciones.

La solución integra:

- 🌐 Webhooks para recibir solicitudes.
- ⚙️ Python para implementar la lógica de negocio.
- 🔀 IF para enrutar el flujo.
- 🗄️ Data Tables para almacenar registros.
- 📧 Gmail para enviar notificaciones.
- 🔄 Respond to Webhook para responder al sistema cliente.

El resultado es un flujo que permite separar la recepción de datos, la evaluación y las acciones posteriores, facilitando la comprensión de los conceptos de automatización e integración de servicios.
