# 💳 Evaluador de Solicitudes de Crédito en Appsmith

## 🎯 Objetivo

Construir una interfaz interactiva y reactiva en **Appsmith** que permita:

- Capturar solicitudes de crédito.
- Ejecutar reglas de evaluación mediante JavaScript.
- Determinar automáticamente el estado de una solicitud.
- Clasificar el nivel de riesgo.
- Mantener un historial dinámico de solicitudes.
- Mostrar los resultados de la evaluación en una tabla.
- Simular una verificación mediante una API externa.
- Permitir la aprobación o el rechazo manual de solicitudes que requieran revisión.

---

## 📐 Arquitectura de la aplicación

La aplicación tendrá los siguientes componentes:

```text
┌─────────────────────────────────────┐
│       Evaluador de Créditos         │
├─────────────────────────────────────┤
│ Solicitante: [________________]     │
│ Monto:       [________________]     │
│ Score:       [________________]     │
│                                     │
│          [Evaluar solicitud]        │
├─────────────────────────────────────┤
│        Historial de solicitudes     │
│                                     │
│ ID | Solicitante | Monto | Estado   │
│ ...                                │
│                                     │
│ Seleccionar fila para revisar       │
└──────────────────┬──────────────────┘
                   │
                   ▼
        ┌────────────────────────┐
        │ Modal de revisión      │
        │                        │
        │ Datos de la solicitud  │
        │ Motivo de evaluación   │
        │                        │
        │ [Aprobar] [Rechazar]   │
        └────────────────────────┘
```

## 🛠️ Configuración paso a paso

### Paso 1. Crear una cuenta y una aplicación

1. Ingresa a [https://www.appsmith.com/](https://www.appsmith.com/).
2. Regístrate o inicia sesión.
3. Selecciona la opción para utilizar Appsmith Cloud.
4. Desde el panel principal, crea una nueva aplicación.
5. Asigna el nombre:

   `Evaluador de Solicitudes de Crédito`

6. Abre el editor de la aplicación.

### Paso 2. Crear la estructura visual

#### 2.1. Agregar un contenedor

1. En el panel de widgets, busca **Container**.
2. Arrástralo al lienzo.
3. Ajusta su tamaño para que contenga el formulario y los demás componentes.

#### 2.2. Agregar el título

1. Arrastra un widget **Text** al contenedor.
2. Configura el texto como:

   `Evaluador de Solicitudes de Crédito`

3. Ajusta el tamaño y el estilo para diferenciarlo del resto de la interfaz.

### Paso 3. Construir el formulario de solicitud

Dentro del contenedor, agrega tres widgets Input y un botón.

| Widget | Nombre | Tipo | Uso |
|---|---|---|---|
| Input | `inp_solicitante` | Text | Nombre del solicitante |
| Input | `inp_monto` | Number | Monto solicitado |
| Input | `inp_score` | Number | Puntaje crediticio |
| Button | `btn_evaluar` | Button | Ejecutar la evaluación |

#### 3.1. Configurar el campo Solicitante

1. Arrastra un widget **Input** al contenedor.
2. En la propiedad **Widget Name**, escribe `inp_solicitante`.
3. Configura la etiqueta como `Nombre del solicitante`.
4. En Placeholder, escribe `Ej. Aura Alba`.

#### 3.2. Configurar el campo Monto

1. Agrega otro widget **Input**.
2. Asigna el nombre `inp_monto`.
3. Selecciona el tipo numérico, si está disponible en la versión del widget.
4. Configura la etiqueta como `Monto solicitado`.
5. Utiliza un valor de ejemplo como `15000`.

#### 3.3. Configurar el campo Score

1. Agrega otro widget **Input**.
2. Asigna el nombre `inp_score`.
3. Configura el tipo numérico.
4. Utiliza la etiqueta `Score crediticio`.
5. Para las pruebas, utiliza valores entre 0 y 1000.

#### 3.4. Crear el botón de evaluación

1. Arrastra un widget **Button**.
2. Asigna el nombre `btn_evaluar`.
3. Configura el texto visible como `Evaluar solicitud`.

El botón ejecutará la lógica de negocio que se creará en el siguiente paso.

---

## 🧠 Paso 4. Crear la lógica de negocio con un JS Object

El motor de evaluación centralizará las reglas del negocio, el almacenamiento del historial y la actualización manual de los estados.

### 4.1. Crear el JS Object

1. En el menú lateral, busca **JS Objects**.
2. Haz clic en **+ New JS Object**.
3. Asigna el nombre:

   `EvaluadorEngine`

4. Reemplaza el contenido inicial con el siguiente código.

### 4.2. Código del motor de evaluación

```javascript
export default {
  UMBRALES: {
    MONTO_MAX_AUTO: 50000,
    SCORE_MINIMO: 600,
    SCORE_EXCELENTE: 750
  },

  obtenerHistorial: () => {
    return appsmith.store.solicitudes || [];
  },

  procesarSolicitud: async () => {
    try {
      const solicitante = inp_solicitante.text;
      const monto = Number(inp_monto.text);
      const score = Number(inp_score.text);

      // Validar los campos
      if (!solicitante || !solicitante.trim()) {
        showAlert("Ingresa el nombre del solicitante.", "warning");
        return;
      }

      if (
        inp_monto.text === "" ||
        !Number.isFinite(monto) ||
        monto <= 0
      ) {
        showAlert("El monto debe ser mayor a 0.", "warning");
        return;
      }

      if (
        inp_score.text === "" ||
        !Number.isFinite(score) ||
        score < 0 ||
        score > 1000
      ) {
        showAlert(
          "El score debe estar entre 0 y 1000.",
          "warning"
        );
        return;
      }

      // Simulación de verificación mediante API externa
      const verificadoExternamente = true;

      const resultado = EvaluadorEngine.evaluarReglas(
        monto,
        score,
        verificadoExternamente
      );

      const nuevoRegistro = {
        id: "SOL-" + Date.now(),
        fecha: new Date().toLocaleString(),
        solicitante: solicitante.trim(),
        monto: monto,
        score: score,
        estado: resultado.estado,
        categoriaRiesgo: resultado.categoriaRiesgo,
        motivo: resultado.motivo,
        verificadoAPI: verificadoExternamente
      };

      const historialActual = EvaluadorEngine.obtenerHistorial();

      const nuevoHistorial = [
        nuevoRegistro,
        ...historialActual
      ];

      await storeValue("solicitudes", nuevoHistorial);

      EvaluadorEngine.notificarResultado(resultado);

      // Limpiar los campos
      resetWidget("inp_solicitante");
      resetWidget("inp_monto");
      resetWidget("inp_score");

    } catch (error) {
      showAlert(
        "Error durante la ejecución: " + error.message,
        "error"
      );
    }
  },

  evaluarReglas: (monto, score, verificado) => {
    // Regla 1: Verificación externa
    if (!verificado) {
      return {
        estado: "RECHAZADO",
        categoriaRiesgo: "ALTO",
        motivo: "Fallo de verificación externa."
      };
    }

    // Regla 2: Validación del monto
    if (monto <= 0) {
      return {
        estado: "RECHAZADO",
        categoriaRiesgo: "INVALIDO",
        motivo: "El monto debe ser mayor a 0."
      };
    }

    // Regla 3: Monto alto y score no excelente
    if (
      monto > EvaluadorEngine.UMBRALES.MONTO_MAX_AUTO &&
      score < EvaluadorEngine.UMBRALES.SCORE_EXCELENTE
    ) {
      return {
        estado: "RECHAZADO",
        categoriaRiesgo: "ALTO",
        motivo: "Exceso de monto para el score actual."
      };
    }

    // Regla 4: Score inferior al mínimo
    if (score < EvaluadorEngine.UMBRALES.SCORE_MINIMO) {
      return {
        estado: "RECHAZADO",
        categoriaRiesgo: "ALTO",
        motivo: "Score por debajo del umbral mínimo."
      };
    }

    // Regla 5: Monto medio y score no excelente
    if (
      monto > 20000 &&
      score < EvaluadorEngine.UMBRALES.SCORE_EXCELENTE
    ) {
      return {
        estado: "REVISION_MANUAL",
        categoriaRiesgo: "MEDIO",
        motivo: "Requiere validación por comité."
      };
    }

    // Regla 6: Aprobación automática
    return {
      estado: "APROBADO",
      categoriaRiesgo: "BAJO",
      motivo: "Aprobación automática cumplida."
    };
  },

  notificarResultado: (resultado) => {
    if (resultado.estado === "APROBADO") {
      showAlert(
        `Aprobado exitosamente (Riesgo ${resultado.categoriaRiesgo})`,
        "success"
      );
    } else if (resultado.estado === "REVISION_MANUAL") {
      showAlert(
        `Enviado a revisión: ${resultado.motivo}`,
        "info"
      );
    } else {
      showAlert(
        `Solicitud rechazada: ${resultado.motivo}`,
        "error"
      );
    }
  },

  cambiarEstado: async (nuevoEstado) => {
    const seleccion = tbl_solicitudes.selectedRow;

    if (!seleccion || !seleccion.id) {
      showAlert("Selecciona una solicitud válida.", "warning");
      return;
    }

    if (
      nuevoEstado !== "APROBADO_MANUAL" &&
      nuevoEstado !== "RECHAZADO"
    ) {
      showAlert("Estado no válido.", "error");
      return;
    }

    const historialActual = EvaluadorEngine.obtenerHistorial();

    const historialActualizado = historialActual.map((solicitud) => {
      if (solicitud.id !== seleccion.id) {
        return solicitud;
      }

      return {
        ...solicitud,
        estado: nuevoEstado,
        categoriaRiesgo:
          nuevoEstado === "APROBADO_MANUAL"
            ? solicitud.categoriaRiesgo
            : "ALTO",
        motivo:
          nuevoEstado === "APROBADO_MANUAL"
            ? "Solicitud aprobada manualmente."
            : "Solicitud rechazada manualmente.",
        fechaActualizacion: new Date().toLocaleString()
      };
    });

    await storeValue("solicitudes", historialActualizado);

    showAlert(
      nuevoEstado === "APROBADO_MANUAL"
        ? "Solicitud aprobada manualmente."
        : "Solicitud rechazada.",
      nuevoEstado === "APROBADO_MANUAL" ? "success" : "info"
    );

    closeModal("mdl_detalle");
  },

  limpiarHistorial: async () => {
    await storeValue("solicitudes", []);

    showAlert("Historial reiniciado.", "info");
  }
};
```

### 4.3. ¿Qué hace el motor?

El código aplica las reglas en el siguiente orden:

| Prioridad | Condición | Resultado |
|---|---|---|
| 1 | Falla la verificación externa | Rechazado, riesgo alto |
| 2 | Monto menor o igual a 0 | Rechazado, datos inválidos |
| 3 | Monto superior a 50.000 y score inferior a 750 | Rechazado, riesgo alto |
| 4 | Score inferior a 600 | Rechazado, riesgo alto |
| 5 | Monto superior a 20.000 y score inferior a 750 | Revisión manual, riesgo medio |
| 6 | Ninguna condición anterior se cumple | Aprobado, riesgo bajo |

**Nota:** la verificación externa está simulada mediante `const verificadoExternamente = true`. No se está consultando una API real.

---

## 🔘 Paso 5. Configurar el botón de evaluación

1. Selecciona el botón `btn_evaluar`.
2. En el panel derecho, localiza **Events**.
3. En el evento `onClick`, selecciona la ejecución de una función JavaScript.
4. Escribe:

```javascript
{{ EvaluadorEngine.procesarSolicitud() }}
```

Cuando el usuario haga clic, Appsmith validará los campos, evaluará la solicitud, almacenará el resultado y mostrará una notificación.

---

## 📊 Paso 6. Crear la tabla interactiva

La tabla permitirá consultar todas las solicitudes evaluadas.

### 6.1. Agregar el widget Table

1. Arrastra un widget **Table** al lienzo, debajo del formulario.
2. Asigna el nombre:

   `tbl_solicitudes`

3. En la propiedad **Table Data**, escribe:

```javascript
{{ EvaluadorEngine.obtenerHistorial() }}
```

### 6.2. Configurar las columnas

Verifica que la tabla muestre las siguientes propiedades:

| Columna | Campo |
|---|---|
| ID | `id` |
| Fecha | `fecha` |
| Solicitante | `solicitante` |
| Monto | `monto` |
| Score | `score` |
| Estado | `estado` |
| Categoría de riesgo | `categoriaRiesgo` |
| Motivo | `motivo` |

Puedes modificar las etiquetas visibles desde la configuración de columnas del widget.

### 6.3. Mostrar los estados visualmente

Si la versión del widget lo permite, configura estilos condicionales para la columna `estado`:

- `APROBADO`: verde.
- `APROBADO_MANUAL`: verde.
- `REVISION_MANUAL`: amarillo.
- `RECHAZADO`: rojo.

Esto permitirá reconocer rápidamente el resultado de cada evaluación.

---

## 🪟 Paso 7. Crear el modal de revisión manual

El modal permitirá revisar una solicitud y registrar una decisión manual.

### 7.1. Crear el modal

1. Agrega un widget **Modal** o crea un modal desde la opción correspondiente de la interfaz.
2. Asigna el nombre:

   `mdl_detalle`

3. Dentro del modal, agrega widgets Text para mostrar los datos de la solicitud seleccionada.

Configura sus valores mediante expresiones:

**Solicitante**

```javascript
{{ tbl_solicitudes.selectedRow.solicitante }}
```

**Monto**

```javascript
{{ tbl_solicitudes.selectedRow.monto }}
```

**Score**

```javascript
{{ tbl_solicitudes.selectedRow.score }}
```

**Estado actual**

```javascript
{{ tbl_solicitudes.selectedRow.estado }}
```

**Motivo**

```javascript
{{ tbl_solicitudes.selectedRow.motivo }}
```

### 7.2. Abrir el modal desde la tabla

1. Selecciona `tbl_solicitudes`.
2. Busca el evento `onRowSelected`.
3. Configura la acción para abrir el modal `mdl_detalle`.

La acción suele estar disponible como **Show Modal** o mediante una función JavaScript, dependiendo de la versión de Appsmith.

### 7.3. Crear el botón Aprobar manualmente

1. Agrega un widget Button dentro de `mdl_detalle`.
2. Asigna el texto `Aprobar manualmente`.
3. En su evento `onClick`, configura:

```javascript
{{ EvaluadorEngine.cambiarEstado('APROBADO_MANUAL') }}
```

Cuando se ejecute, el estado de la solicitud seleccionada cambiará a `APROBADO_MANUAL` y se actualizará el historial.

### 7.4. Crear el botón Rechazar

1. Agrega otro widget Button dentro del modal.
2. Asigna el texto `Rechazar`.
3. En su evento `onClick`, configura:

```javascript
{{ EvaluadorEngine.cambiarEstado('RECHAZADO') }}
```

La solicitud seleccionada quedará registrada como rechazada manualmente.

**Importante:** estas acciones actualizan el estado guardado en Appsmith, pero no representan una aprobación financiera real ni notifican a sistemas externos.

---

## 🧹 Paso 8. Agregar un botón para limpiar el historial

1. Agrega un botón llamado `btn_limpiar`.
2. Configura su texto como `Limpiar historial`.
3. En el evento `onClick`, utiliza:

```javascript
{{ EvaluadorEngine.limpiarHistorial() }}
```

Esta acción elimina los registros de la colección `solicitudes` almacenada en `appsmith.store`.

Utilízala únicamente durante las pruebas, ya que vacía el historial actual.

---

## 🧪 Paso 9. Probar el evaluador

Prueba los siguientes casos para verificar las reglas de negocio.

| Solicitante | Monto | Score | Resultado esperado |
|---|---:|---:|---|
| Ana Pérez | 15.000 | 800 | `APROBADO` |
| Carlos Ruiz | 60.000 | 700 | `REVISION_MANUAL` |
| Laura Gómez | 60.000 | 650 | `RECHAZADO` |
| Pedro Díaz | 25.000 | 550 | `RECHAZADO` |
| María López | 30.000 | 700 | `REVISION_MANUAL` |

Para cada caso:

1. Completa los campos del formulario.
2. Haz clic en `Evaluar solicitud`.
3. Verifica el mensaje que aparece.
4. Comprueba que el registro se agregue a la tabla.
5. Selecciona una fila para abrir el modal.
6. Prueba las acciones de aprobación manual y rechazo.
7. Verifica que el estado y el motivo se actualicen.

---

## 💾 Consideraciones sobre el almacenamiento

La aplicación utiliza `appsmith.store` para mantener el historial durante la sesión de trabajo.

Este mecanismo es útil para una demostración y para compartir estado entre widgets, pero **no sustituye una base de datos persistente**. Para una solución de producción, se recomienda conectar Appsmith con una base de datos o un servicio backend que conserve los registros, controle los permisos y mantenga la trazabilidad de las decisiones.

## 📌 Conclusión

Este proyecto demuestra cómo construir una interfaz de evaluación de solicitudes de crédito mediante Appsmith, combinando componentes visuales, JavaScript y almacenamiento de estado.

La solución integra:

- 📝 Un formulario para capturar solicitudes.
- 🧠 Un motor de reglas de negocio.
- 📊 Una tabla dinámica para consultar el historial.
- 🪟 Un modal para la revisión manual.
- 🔄 Actualización de estados sin recargar la aplicación.
- 💾 Almacenamiento temporal mediante `appsmith.store`.

De esta manera, Appsmith funciona como una capa de interacción que permite a los usuarios registrar solicitudes, consultar sus resultados y participar en el proceso de revisión.
