# Code Review - Semana 8

## Revisor: Edwards Perez | Revisado: Jaiker Figueredo

### Aspectos Positivos
1. **Arquitectura modular de servicios:** La implementación de `src/services/erpApi.js` encapsula adecuadamente Axios mediante `axios.create` con `baseURL`, `timeout` y cabeceras JSON, separando claramente la capa de red de los componentes de vista.
2. **Manejo asíncrono robusto:** La utilización estricta de `try/catch/finally` en `cargarMovimientos` y `guardarMovimiento` asegura que el estado `cargando` siempre se restablezca, evitando bloqueos visuales o spinners congelados.
3. **Manejo defensivo en propiedades computadas:** `totalIngresos`, `totalEgresos` y `saldo` validan correctamente arrays vacíos, garantizando que el saldo devuelva `0` y nunca genere valores `NaN`.

### Sugerencias de Mejora
1. **Notificaciones tipo Toast / SnackBar:** Para una mejor experiencia visual, los mensajes de confirmación de guardado podrían implementarse mediante `<v-snackbar>` además de la alerta contextual.
2. **Interceptores globales de Axios:** Se podría configurar un interceptor de respuesta en `erpApi.js` para registrar errores de autenticación (ej: HTTP 401) o redirigir globalmente en caso de caída del servidor.

### Preguntas Técnicas
1. **¿Por qué es fundamental ejecutar `cargando.value = false` dentro del bloque `finally` en lugar del final del bloque `try`?**
   * *Respuesta:* Porque si ocurre un error dentro de `try`, la ejecución salta inmediatamente al `catch` sin ejecutar el resto del código del `try`. Ubicarlo en `finally` garantiza que el indicador de carga se oculte tanto si la petición tiene éxito como si falla.

### Validación de Funcionalidades
- [x] Carga de movimientos funciona
- [x] Creación de movimientos funciona
- [x] Manejo de errores es adecuado (prueba con servidor apagado)
- [x] KPIs (saldo) se actualizan correctamente
- [x] El código usa async/await y try/catch/finally
