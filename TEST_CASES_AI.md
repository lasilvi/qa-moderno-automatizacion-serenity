# CASOS DE PRUEBA - SISTEMA DE GESTIÓN DE RESTAURANTE

## TABLA DE AJUSTES DEL PROBADOR

| ID | Caso de Prueba Generado por Gema | Ajuste del Probador | Motivo del Ajuste |
|----|----------------------------------|----------------------|-------------------|
| CP-01 | H1 - Validación de número de mesa (Límite inferior). Técnica: Análisis de valores límite. Dado cliente en TableSelectPage.tsx. Cuando ingresa mesa 0. Entonces muestra error y bloquea acceso al menú. | Sin ajuste | El caso está correctamente definido y es verificable. |
| CP-02 | H1 - Selección de mesa válida y persistencia de sesión. Dado mesa 5 seleccionada. Cuando cierra navegador y vuelve. Entonces recupera mesa desde useApp(). | Se especifica que la mesa debe estar previamente persistida en LocalStorage para ser recuperada por el contexto global. | Se elimina ambigüedad sobre el mecanismo real de persistencia. |
| CP-03 | H1 - Filtrado de productos inactivos (isActive=false). Cuando carga MenuPage.tsx. Entonces solo muestra productos activos. | Sin ajuste | El criterio funcional es claro y medible. |
| CP-04 | H1 - Persistencia del carrito en LocalStorage. Cuando refresca o cierra pestaña. Entonces mantiene producto y cantidad. | Sin ajuste | El comportamiento esperado está correctamente delimitado. |
| CP-05 | H1 - Intento de compra con producto desactivado. Backend responde 400 Bad Request y frontend notifica indisponibilidad. | No aplica (Caso excluido) | Caso excluido del alcance funcional definido para la entrega actual. |
| CP-06 | H1 - Confirmación exitosa y publicación de evento order.placed en RabbitMQ. | Se agrega validación de respuesta HTTP 201 Created tras la persistencia exitosa. | Permite validar formalmente el éxito de la operación en backend. |
| CP-07 | H2 - Acceso denegado al tablero sin X-Kitchen-Token. Retorna 401 Unauthorized. | Sin ajuste | Caso de seguridad correctamente definido. |
| CP-08 | H2 - Acceso exitoso con token válido cocina123 y visualización del tablero Kanban. | Sin ajuste | Flujo de autenticación claro y verificable. |
| CP-09 | H2 - Actualización automática del tablero (≤ 3 segundos). | Se especifica que el tiempo se mide desde la persistencia del pedido en base de datos. | Se evita ambigüedad en la medición del tiempo para pruebas de rendimiento. |
| CP-10 | H2 - Transición válida de estado PENDING → IN_PREPARATION. Actualiza estado y campo updatedAt en base de datos. | Sin ajuste | Flujo coherente con el modelo de estados definido. |
| CP-11 | H2 - Transición inválida READY → PENDING. Lanza InvalidStatusTransitionException y retorna 400 Bad Request. | Sin ajuste | Regla de negocio RN-001 correctamente validada. |
| CP-12 | H2 - Eliminación masiva (Soft Delete). Marca pedidos como deleted=true y los oculta de la vista manteniéndolos en BD. | No aplica (Caso excluido) | Funcionalidad no implementada ni incluida dentro del alcance del sprint actual. |
| CP-13 | H3 - Consulta pública del estado del pedido vía UUID sin autenticación. | Sin ajuste | Caso funcional claro y correctamente delimitado. |
| CP-14 | H3 - Representación visual por colores según estado (Amarillo=PENDING, Azul=IN_PREPARATION, Verde=READY). | Se especifica que el cambio de estado es detectado mediante polling del frontend. | Corrección técnica para reflejar el flujo real del sistema. |
| CP-15 | H3 - Detención automática del polling cuando el estado cambia a READY. | No aplica (Caso excluido) | Optimización de eficiencia no contemplada dentro del alcance funcional validado. |
| CP-16 | H3 - Consulta de pedido inexistente o eliminado. Retorna 404 Not Found. | Sin ajuste | Manejo de error correctamente especificado y verificable. |

---

## H1: SELECCIÓN DE MESA Y CREACIÓN DE PEDIDO (CLIENTE)

### Técnicas Aplicadas
- Análisis de valores límite (mesa / cantidad)
- Partición de equivalencia (isActive)
- Pruebas de estado persistente
- Pruebas negativas (manejo de excepciones)
- Validación de eventos asíncronos

### Caso de Prueba 1: Validación de número de mesa (Límite inferior)
**Técnica:** Análisis de valores límite

**Dado que** el cliente se encuentra en la página de selección de mesa `TableSelectPage.tsx`  
**Cuando** ingresa el número de mesa **0**  
**Entonces** el sistema debe mostrar un error de validación y no permitir avanzar al menú

**Resultado esperado:**
- Se muestra mensaje de error
- No hay navegación a `/client/menu`

### Caso de Prueba 2: Selección de mesa válida y persistencia de sesión
**Técnica:** Prueba de estado persistente

**Dado que** el cliente ingresa el número de mesa **5**  
**Cuando** el cliente cierra el navegador y vuelve a ingresar a la aplicación  
**Entonces** el sistema debe recuperar el número de mesa 5 desde el contexto global `useApp()`

**Resultado esperado:**
- La aplicación redirige al menú
- El número de mesa permanece almacenado

### Caso de Prueba 3: Filtrado de productos inactivos en el menú
**Técnica:** Partición de equivalencia (isActive = true / false)

**Dado que** existen productos en la base de datos con `isActive = false`  
**Cuando** el cliente carga la página `MenuPage.tsx`  
**Entonces** el sistema solo debe mostrar los productos donde `isActive = true`

**Resultado esperado:**
- Productos inactivos no visibles en la interfaz
- El endpoint `GET /menu` retorna únicamente productos activos

### Caso de Prueba 4: Persistencia del carrito en LocalStorage
**Técnica:** Prueba de persistencia

**Dado que** el cliente agrega **"Empanadas criollas"** al carrito  
**Cuando** refresca la página o cierra la pestaña del navegador  
**Entonces** al volver a entrar, el carrito debe mantener el producto y la cantidad seleccionada

**Resultado esperado:**
- Datos recuperados desde `localStorage`
- Totales recalculados correctamente

### Caso de Prueba 5: Intento de compra con producto desactivado (Excepción)
**Técnica:** Prueba negativa / Regla **RN-002**

**Dado que** el cliente tiene un producto en el carrito que fue desactivado recientemente  
**Cuando** hace clic en **"Confirmar Pedido"**  
**Entonces** el backend debe responder con `400 Bad Request` y el frontend debe notificar que el producto ya no está disponible

**Resultado esperado:**
- No se crea la orden
- Se muestra mensaje de error claro

### Caso de Prueba 6: Confirmación exitosa y publicación de evento
**Técnica:** Validación funcional + Evento asíncrono

**Dado que** el cliente tiene un carrito válido y una mesa asignada  
**Cuando** presiona el botón **"Confirmar Pedido"**  
**Entonces** el sistema debe:
- Generar un **UUID único**
- Persistir la orden con estado **PENDING**
- Publicar el evento `order.placed` en **RabbitMQ**

**Resultado esperado:**
- Redirección a `/client/confirm/{orderId}`
- Registro visible en base de datos
- Evento publicado correctamente

---

## H2: GESTIÓN DE PEDIDOS (COCINA)

### Técnicas Aplicadas
- Tabla de decisión (Seguridad)
- Transición de estados (Máquina de estados)
- Pruebas de carga ligera (Polling)
- Validación de reglas de negocio (**RN-001**)
- Soft delete

### Caso de Prueba 7: Acceso denegado sin token
**Técnica:** Tabla de decisión (Header presente/ausente)

**Dado que** un usuario intenta acceder a `GET /orders` sin el header `X-Kitchen-Token`  
**Cuando** la solicitud es interceptada por `KitchenSecurityInterceptor`  
**Entonces** el sistema debe retornar `401 Unauthorized`

### Caso de Prueba 8: Acceso exitoso con token válido
**Dado que** el personal configura el token **cocina123**  
**Cuando** accede a `/kitchen/board` enviando el header correcto  
**Entonces** el sistema debe permitir visualizar los pedidos en las columnas del **Kanban**

### Caso de Prueba 9: Validación de actualización automática (Polling)
**Técnica:** Prueba de eficiencia

**Dado que** el personal visualiza el tablero Kanban  
**Cuando** entra un nuevo pedido  
**Entonces** el tablero debe actualizarse automáticamente en un tiempo **no mayor a 3 segundos** sin necesidad de recargar la página

### Caso de Prueba 10: Transición de estado válida
**Técnica:** Máquina de estados

**Dado que** un pedido está en **PENDING**  
**Cuando** el cocinero presiona **"Iniciar"**  
**Entonces** el estado cambia a **IN_PREPARATION** y el campo `updatedAt` se actualiza en la base de datos

### Caso de Prueba 11: Transición inválida (RN-001)
**Dado que** un pedido está en **READY**  
**Cuando** se intenta cambiar vía API a **PENDING**  
**Entonces** el sistema debe lanzar `InvalidStatusTransitionException` y retornar `400 Bad Request`

### Caso de Prueba 12: Eliminación masiva (Soft Delete)
**Técnica:** Prueba de integridad lógica

**Dado que** el personal ejecuta la acción de **"Borrado Masivo"**  
**Cuando** confirma la operación  
**Entonces** el sistema debe:
- Marcar todos los pedidos como `deleted = true`
- Hacerlos desaparecer de la vista
- Mantenerlos en la base de datos para auditoría

---

## H3: CONSULTA DE ESTADO (CLIENTE)

### Técnicas Aplicadas
- Partición de equivalencia (Estados visuales)
- Pruebas de eficiencia (Detención de polling)
- Pruebas negativas (404)

### Caso de Prueba 13: Acceso público al estado vía UUID
**Dado que** el cliente tiene el **UUID** de su pedido generado exitosamente  
**Cuando** ingresa a la URL `/client/status/{orderId}` sin autenticación  
**Entonces** el sistema debe mostrar el resumen del pedido y su estado actual correctamente

### Caso de Prueba 14: Representación visual de estados
**Técnica:** Partición por estado

**Dado que** el cliente está consultando su pedido  
**Cuando** el estado cambia en la base de datos  
**Entonces** el frontend debe mostrar:
- **PENDING** → Amarillo
- **IN_PREPARATION** → Azul
- **READY** → Verde

### Caso de Prueba 15: Detención de Polling
**Técnica:** Optimización de recursos

**Dado que** el cliente está en la página de seguimiento  
**Cuando** el estado cambia a **READY**  
**Entonces** el proceso de polling de **5 segundos** debe detenerse automáticamente

### Caso de Prueba 16: Consulta de pedido inexistente o eliminado
**Técnica:** Prueba negativa

**Dado que** un pedido fue eliminado mediante soft delete o el UUID es inválido  
**Cuando** el cliente intenta acceder a la página de estado  
**Entonces** el sistema debe retornar `404 Not Found` y mostrar un mensaje indicando que el pedido no fue encontrado