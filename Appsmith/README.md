# 💳 Evaluador de Solicitudes de Crédito en Appsmith

## 🎯 Objetivo

Construir una interfaz **interactiva y reactiva en Appsmith** que permita:

- Capturar solicitudes de crédito.
- Ejecutar reglas de evaluación mediante código **JavaScript local**.
- Determinar automáticamente el estado de una solicitud.
- Clasificar el nivel de riesgo.
- Mantener un historial dinámico de solicitudes.
- Mostrar estilos visuales condicionales según el resultado de la evaluación.
- Simular una verificación mediante una API externa.

---

## 📐 Lógica del Motor de Evaluación

El proyecto utiliza un **JS Object** llamado `EvaluadorEngine`, encargado de centralizar la lógica de evaluación de las solicitudes.

### Crear el JS Object

En Appsmith:

**JS Objects → New JS Object → `EvaluadorEngine`**

Agregar la siguiente implementación:

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

      // Simulación de verificación mediante API externa
      const verificadoExternamente = true;

      const resultado = this.evaluarReglas(
        monto,
        score,
        verificadoExternamente
      );

      const nuevoRegistro = {
        id: "SOL-" + Math.floor(1000 + Math.random() * 9000),
        fecha: new Date().toLocaleTimeString(),
        solicitante: solicitante,
        monto: monto,
        score: score,
        estado: resultado.estado,
        categoriaRiesgo: resultado.categoriaRiesgo,
        motivo: resultado.motivo,
        verificadoAPI: verificadoExternamente
      };

      const historialActual = this.obtenerHistorial();
      const nuevoHistorial = [
        nuevoRegistro,
        ...historialActual
      ];

      await storeValue("solicitudes", nuevoHistorial);

      this.notificarResultado(resultado);

      // Limpiar formulario
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

    // Regla 3: Monto alto + score no excelente
    if (
      monto > this.UMBRALES.MONTO_MAX_AUTO &&
      score < this.UMBRALES.SCORE_EXCELENTE
    ) {
      return {
        estado: "RECHAZADO",
        categoriaRiesgo: "ALTO",
        motivo: "Exceso de monto para el score actual."
      };
    }

    // Regla 4: Score inferior al mínimo permitido
    if (score < this.UMBRALES.SCORE_MINIMO) {
      return {
        estado: "RECHAZADO",
        categoriaRiesgo: "ALTO",
        motivo: "Score por debajo del umbral mínimo."
      };
    }

    // Regla 5: Monto medio/alto + score no excelente
    if (
      monto > 20000 &&
      score < this.UMBRALES.SCORE_EXCELENTE
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
        `Solicitud Rechazada: ${resultado.motivo}`,
        "error"
      );
    }
  },

  limpiarHistorial: async () => {

    await storeValue("solicitudes", []);

    showAlert(
      "Historial reiniciado.",
      "info"
    );
  }
};
```

---

# 🛠️ Configuración de la Interfaz

## 1. Formulario de Solicitud

Crear un formulario denominado:

`Form1`

El formulario debe contener los siguientes componentes:

| Componente | Nombre | Tipo |
|---|---|---|
| Input | `inp_solicitante` | Text |
| Input | `inp_monto` | Number |
| Input | `inp_score` | Number |
| Botón Submit | — | Button |

### Configuración de los Inputs

#### Solicitante

- **Widget:** Input
- **Name:** `inp_solicitante`
- **Tipo:** Text
- **Uso:** Nombre del solicitante.

#### Monto

- **Widget:** Input
- **Name:** `inp_monto`
- **Tipo:** Number
- **Uso:** Valor solicitado del crédito.

#### Score

- **Widget:** Input
- **Name:** `inp_score`
- **Tipo:** Number
- **Uso:** Puntaje crediticio del solicitante.

---

## 2. Botón de envío

El botón **Submit** debe ejecutar el método:

```javascript
EvaluadorEngine.procesarSolicitud()
```

### Evento

**onClick → Execute a JS function**

```javascript
EvaluadorEngine.procesarSolicitud()
```

Al hacer clic:

1. Se obtienen los datos del formulario.
2. Se convierten monto y score a números.
3. Se ejecutan las reglas de evaluación.
4. Se genera