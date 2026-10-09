# 🛠️ Taller Práctico: Ecosistemas Low-Code — Automatización Backend vs. Interfaces Frontend

## 🎯 Objetivo del Taller

Construir la misma lógica de negocio en dos entornos Low-Code diferentes (**n8n y Appsmith**) para comparar su arquitectura, experiencia de desarrollo, facilidad de uso y casos de aplicación ideales.

## 📌 Escenario de Negocio

El departamento de Operaciones necesita un sistema para **evaluar y procesar solicitudes de descuento para clientes**.

El sistema debe aplicar las siguientes reglas de negocio:

1. **Aprobación automática:** si el porcentaje de descuento solicitado es menor o igual al 15 %.
2. **Aprobación gerencial:** si el porcentaje de descuento solicitado es superior al 15 %, la solicitud pasa al estado `REQUIERE_APROBACION_GERENCIAL`.
3. **Rechazo por exceso:** si el valor monetario del descuento supera los $5.000, la solicitud se marca como `RECHAZADO_EXCESIVO`.

> **Importante:** Las reglas se evalúan en orden de prioridad. El rechazo por exceso tiene prioridad sobre la aprobación automática y la aprobación gerencial.

---

# ⚙️ Parte 1: Implementación en n8n

**Enfoque:** Event-Driven / API — Backend & Workflows.

### 🎯 Objetivo

Crear un microservicio mediante un Webhook que reciba una solicitud en formato JSON, evalúe las reglas de negocio y devuelva una respuesta con la decisión procesada.

## 1. Crear el Webhook de entrada

Crear un nuevo workflow en n8n y agregar un nodo **Webhook**.

Configurar las siguientes propiedades:

| Propiedad | Valor |
|---|---|
| HTTP Method | `POST` |
| Path | `evaluar-descuento` |
| Respond | `Using 'Respond to Webhook' Node` |

El Webhook será el punto de entrada para recibir las solicitudes de evaluación.

### Ejemplo de payload

```json
{
  "cliente": "Cliente Demo",
  "monto": 40000,
  "descuento_pct": 10
}
```

## 2. Implementar la lógica de negocio

Agrega un nodo **Code** y selecciona el lenguaje Python.

Requerimiento del código: Escribe la lógica necesaria utilizando la función nativa get_input_data() de n8n para:

- Extraer del JSON de entrada los campos: cliente (String), monto (Numérico) y descuento_pct (Numérico).

- Calcular el valor del descuento en dinero según la fórmula dada.

- Evaluar las 3 reglas de negocio y asignar la propiedad estado.

- Retornar una estructura con la clave "json" que incluya los datos de entrada, el valor calculado del descuento y el estado final.

## 3. Devolver la respuesta HTTP

Agregar un nodo **Respond to Webhook** al final del flujo.

Configurar el nodo para devolver el resultado de la evaluación en formato JSON.

Ejemplo de respuesta:

```json
{
  "cliente": "Cliente Demo",
  "monto": 40000,
  "descuento_pct": 10,
  "valor_descuento": 4000,
  "estado": "APROBADO_AUTOMATICO"
}
```

El objetivo es que el cliente que realizó la petición HTTP reciba el resultado del procesamiento.

## 4. Probar el flujo

Ejecutar el workflow y enviar una solicitud desde Postman o cURL.

```bash
curl -X POST "<TU_WEBHOOK_URL>" \
  -H "Content-Type: application/json" \
  -d '{
    "cliente": "Cliente Demo",
    "monto": 40000,
    "descuento_pct": 10
  }'
```

Reemplaza `<TU_WEBHOOK_URL>` por la URL de prueba o producción del Webhook configurado en n8n.

---

# 🖥️ Parte 2: Implementación en Appsmith

**Enfoque:** UI / Reactivo — Frontend & State Management.

### 🎯 Objetivo

Crear un formulario para capturar solicitudes de descuento y una tabla para visualizar los resultados de las evaluaciones.

## 1. Diseñar la interfaz

Crear una aplicación en Appsmith con los siguientes componentes:

| Componente | Nombre | Tipo |
|---|---|---|
| Cliente | `txt_cliente` | Input de texto |
| Monto de venta | `num_monto` | Input numérico |
| Porcentaje de descuento | `num_descuento` | Input numérico |
| Evaluar descuento | — | Button |
| Historial de solicitudes | `tabla_historial` | Table |

La interfaz debe permitir ingresar los datos, ejecutar la evaluación y consultar los registros generados.

## 2. Crear el JS Object

En el panel izquierdo, crea un New JS Object con el nombre EvaluadorDescuentos.

**Requerimiento de la función JS:** Escribe una función asíncrona (procesar) que realice lo siguiente:

-Validar que los campos de la interfaz no estén vacíos (mostrar una alerta si falta algún dato).

- Leer los valores de los widgets txt_cliente, num_monto y num_descuento.

- Ejecutar la misma lógica de negocio descrita en el escenario para determinar el estado y el valorDescuento.

- Crear un nuevo objeto de registro que incluya los datos procesados, un identificador único (ej. Date.now()) y la hora de registro.

- Guardar el nuevo registro en el almacenamiento local de Appsmith usando storeValue('historial', ...) para mantener la persistencia durante la sesión.

- Limpiar los inputs del formulario y mostrar una alerta flotante (showAlert) informando el resultado.

## 3. Vincular la interfaz

Configura el evento onClick del botón para invocar tu función JavaScript.

Conecta la propiedad Table Data de la tabla con el arreglo almacenado en tu objeto JS mediante expresiones corchete ({{ ... }}).

La tabla mostrará las solicitudes procesadas y su estado correspondiente.

> **Nota:** `appsmith.store` permite almacenar datos en el estado de la aplicación. No sustituye una base de datos persistente y compartida para un sistema de producción. Si se necesita persistencia duradera, debe utilizarse una fuente de datos apropiada.

---

# 🔗 Parte 3: Reto de Integración OPCIONAL

**Enfoque:** Análisis e Integración.

**Duración estimada:** 10 minutos.

### 🎯 Objetivo

Integrar Appsmith con n8n para que la interfaz utilice el Webhook como servicio de evaluación, en lugar de ejecutar localmente el JS Object.

## Pasos de implementación

1. Mantener el formulario de captura de datos en Appsmith.
2. Crear una consulta de tipo **API REST** con método `POST`.
3. Configurar la URL del Webhook de n8n.
4. Enviar los datos del formulario en formato JSON.
5. Recibir la respuesta de n8n.
6. Mostrar el estado devuelto en la interfaz.
7. Actualizar el historial con el resultado recibido.

### Ejemplo de cuerpo de la petición

```json
{
  "cliente": "{{txt_cliente.text}}",
  "monto": "{{Number(num_monto.text)}}",
  "descuento_pct": "{{Number(num_descuento.text)}}"
}
```

Este ejemplo ilustra los datos que deben enviarse. En Appsmith, configura el cuerpo de la consulta utilizando sus expresiones de binding y un objeto JSON válido.

### Arquitectura de integración

```text
┌──────────────────────────┐
│        Appsmith          │
│                          │
│ Formulario de descuentos │
└────────────┬─────────────┘
             │
             │ HTTP POST
             ▼
┌──────────────────────────┐
│          n8n             │
│                          │
│ Webhook                  │
│ Code - Reglas de negocio │
│ Respond to Webhook       │
└────────────┬─────────────┘
             │
             │ Respuesta JSON
             ▼
┌──────────────────────────┐
│        Appsmith          │
│                          │
│ Mostrar resultado        │
│ Actualizar historial     │
└──────────────────────────┘
```

### Resultado esperado

La integración permite combinar las fortalezas de ambas herramientas:

- **Appsmith:** captura de datos, visualización de resultados e interacción con el usuario.
- **n8n:** procesamiento centralizado, automatización e integración con servicios externos.

Así, ambas plataformas pasan de ser alternativas para resolver el mismo problema a ser componentes complementarios de una solución.

---

# 🧪 Casos de Prueba

Los siguientes escenarios permiten verificar que las reglas de negocio se ejecutan en el orden esperado.

| Caso | Monto | Descuento | Valor del descuento | Resultado esperado |
|---|---:|---:|---:|---|
| 1 | $40.000 | 10 % | $4.000 | `APROBADO_AUTOMATICO` |
| 2 | $40.000 | 20 % | $8.000 | `RECHAZADO_EXCESIVO` |
| 3 | $20.000 | 20 % | $4.000 | `REQUIERE_APROBACION_GERENCIAL` |
| 4 | $50.000 | 10 % | $5.000 | `APROBADO_AUTOMATICO` |
| 5 | $60.000 | 15 % | $9.000 | `RECHAZADO_EXCESIVO` |

**Observación:** Un descuento de exactamente $5.000 no se rechaza por exceso, porque la regla exige que el valor sea estrictamente superior a $5.000.

---

# ❓ Cuestionario de Reflexión y Comparación

Al finalizar el ejercicio, responder las siguientes preguntas.

## 1. Experiencia de Desarrollo y Curva de Aprendizaje

- ¿En cuál de las dos herramientas resultó más intuitivo modelar la regla de negocio y por qué?
- ¿Cómo afectó la forma de trabajar con variables (Python en n8n frente a JavaScript en Appsmith)?
- ¿Qué diferencias se identificaron al depurar errores y probar la lógica?

## 2. Arquitectura y Enfoque de Uso

- Si una empresa necesita automatizar un proceso que se ejecuta todas las noches sin interacción humana —por ejemplo, sincronizar 10.000 registros entre dos bases de datos—, ¿cuál herramienta elegirías y por qué?
- Si un equipo de Operaciones necesita un panel interactivo para aprobar o rechazar solicitudes manualmente y enviar notificaciones, ¿cuál herramienta utilizarías como capa principal?
- ¿Qué diferencias existen entre automatizar procesos y construir interfaces para usuarios?

## 3. Gestión de Datos y Persistencia

- ¿Qué alternativa utilizarías si el historial debe conservarse de manera duradera y compartirse entre múltiples usuarios?
- ¿Cómo influye la persistencia de los datos en la confiabilidad de la solución?

---

# 📋 Entregables
Documento con los screenshoots de los flujos diseñados en las herramientas n8n y Appsmith.
Agregar la respuesta a las preguntas propuestas de comparación.

---

# 📌 Conclusiones

Este taller permite comparar dos enfoques Low-Code complementarios:

- **n8n** facilita la automatización de procesos orientados a eventos, la integración entre sistemas y la ejecución de reglas de negocio mediante workflows.
- **Appsmith** facilita la creación de interfaces internas, formularios interactivos, paneles administrativos y herramientas de gestión.
- **La integración de ambas plataformas** permite separar la experiencia de usuario del procesamiento de negocio, favoreciendo la reutilización de la lógica y la evolución de la solución.

La elección depende del problema que se desea resolver, los requisitos de persistencia, la interacción humana, las integraciones necesarias y las condiciones operativas del sistema.
