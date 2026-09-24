# 📝 Fase A: Informe de Investigación Asíncrona (Semana 8)
**Estudiante:** Estudiante  
**Materia:** Programación Web Avanzada / Arquitectura Frontend-Backend  
**Proyecto:** ERP Contable (Vue 3 + Express)

---

## 1. Escenario 1: Simulación de Servidor Caído / Falla de Red

### ❓ Pregunta:
*¿Qué tipo de error se genera en la consola si el servidor Express está apagado o inaccesible? ¿Cómo lo gestiona y muestra la interfaz de usuario (UI)?*

### 💡 Análisis y Respuesta:
* **Error en Consola y Red:** Al estar apagado el servidor Express (`http://localhost:3000`), el navegador no puede establecer conexión TCP (`ERR_CONNECTION_REFUSED` o `AxiosError: Network Error` con código `ERR_NETWORK`).
* **Comportamiento Asíncrono (`try/catch/finally`):**
  1. En el bloque `try`, `await movimientoService.getAll()` rechaza la promesa lanzando una excepción.
  2. El bloque `catch (err)` captura el error y asigna un mensaje descriptivo y amigable a la variable reactiva `error.value`:  
     `"No se pudo conectar con el servidor Express (http://localhost:3000). Verifica que esté encendido."`
  3. El bloque `finally` se ejecuta incondicionalmente, estableciendo `cargando.value = false`. Esto **evita que el spinner de carga se quede congelado indefinidamente**.
* **Visualización en UI:** En la pantalla se renderiza un componente `<v-alert type="error">` con el mensaje claro para el usuario, manteniendo la aplicación responsiva e interactiva sin romperse.

---

## 2. Escenario 2: Manejo de Errores HTTP 400 (Validación en Backend)

### ❓ Pregunta:
*¿Cómo se atrapa el error cuando se intenta enviar datos inválidos (como un monto negativo o campos vacíos) y cómo se muestra el mensaje exacto devuelto por la API?*

### 💡 Análisis y Respuesta:
* **Respuesta del Servidor:** Cuando se envía un `POST /api/movimientos` con un monto `<= 0`, el backend de Express responde con un código de estado `HTTP 400 Bad Request` y un JSON estructurado:
  ```json
  {
    "exito": false,
    "mensaje": "El monto debe ser un número positivo"
  }
  ```
* **Captura en Axios (`err.response.data`):**
  Axios trata cualquier código fuera del rango 2xx como un error en la promesa. Dentro del `catch (err)`, accedemos de forma segura al mensaje personalizado del backend mediante:
  ```javascript
  error.value = err.response?.data?.mensaje || 'Error al procesar la solicitud.';
  ```
* **Visualización en UI:** La interfaz muestra dinámicamente el mensaje exacto que envió el servidor en la alerta de Vuetify, guiando al usuario para que corrija el valor ingresado.

---

## 3. Escenario 3: Carga Concurrente con `Promise.all`

### ❓ Pregunta:
*¿Cómo se implementa la carga simultánea de múltiples recursos independientes (`contactos` y `movimientos`) utilizando `Promise.all` y cuáles son sus ventajas frente a llamadas secuenciales?*

### 💡 Implementación y Explicación:
Si hiciéramos dos llamadas secuenciales con `await`, la segunda tendría que esperar a que termine la primera:
```javascript
// Secuencial (Más lento: Tiempo A + Tiempo B)
const res1 = await contactoService.getAll();
const res2 = await movimientoService.getAll();
```

Con `Promise.all`, ambas peticiones se disparan en paralelo y esperamos a que ambas se resuelvan concurrentemente:
```javascript
async function cargarDatosGlobales() {
  cargando.value = true;
  error.value = null;

  try {
    // Disparo simultáneo de ambas promesas
    const [resContactos, resMovimientos] = await Promise.all([
      contactoService.getAll(),
      movimientoService.getAll()
    ]);

    contactos.value = resContactos.data.datos;
    movimientos.value = resMovimientos.data.datos;
  } catch (err) {
    error.value = 'Ocurrió un error al cargar los datos del sistema.';
    console.error(err);
  } finally {
    // El spinner solo se oculta cuando ambas peticiones han concluido (o una falló)
    cargando.value = false;
  }
}
```

### 🚀 Ventajas:
1. **Rendimiento Óptimo (Menor Latencia):** El tiempo total de espera es el tiempo de la petición más lenta (`max(t1, t2)`), en lugar de la suma de ambas (`t1 + t2`).
2. **Experiencia de Usuario Fluida:** La interfaz pasa del estado de carga al estado listo en una sola transición unificada.
3. **Manejo Centralizado de Errores:** Si cualquiera de las dos promesas falla, el bloque `catch` reacciona de inmediato para informar el problema.
