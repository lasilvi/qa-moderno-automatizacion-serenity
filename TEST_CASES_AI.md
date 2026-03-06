# CASOS DE PRUEBA - SISTEMA DE GESTIÓN DE RESTAURANTE

## TABLA DE AJUSTES DEL PROBADOR

| ID | Caso de Prueba Generado por Gema | Ajuste del Probador | Motivo del Ajuste |
|----|----------------------------------|----------------------|-------------------|
| CP-01 | Validación de número de mesa (valor límite inferior: 0). | Sin ajuste | Caso válido para validar restricción de dominio (mesa ≥ 1). |
| CP-02 | Acceso exitoso al menú mediante selección de mesa válida (mesa = 1). | Sin ajuste | Flujo funcional básico correcto. |
| CP-03 | Restricción de cambio de mesa en la misma sesión. | Eliminado | No existe funcionalidad implementada que restrinja el cambio de mesa en sesión. |
| CP-04 | Persistencia del carrito al cerrar o refrescar el navegador. | Sin ajuste | Comportamiento esperado mediante LocalStorage/CartContext. |
| CP-05 | Filtrado de productos inactivos en el menú. | Eliminado | La funcionalidad de gestión de productos activos/inactivos no está implementada actualmente en el sistema. |
| CP-06 | Desactivación de producto durante la conformación del pedido. | Eliminado | Funcionalidad dependiente de administración de productos no implementada en el alcance actual. |
| CP-07 | Confirmación exitosa del pedido y generación de UUID. | Se agrega validación HTTP 201 Created | Permite validar formalmente la respuesta correcta del backend. |
| CP-08 | Visualización del tablero de cocina ordenado por prioridad temporal. | Sin ajuste | Flujo esperado de visualización de pedidos en cocina. |
| CP-09 | Transición de estado PENDING → IN_PREPARATION. | Sin ajuste | Flujo válido dentro de la máquina de estados del pedido. |
| CP-10 | Finalización del pedido y publicación de evento en RabbitMQ. | Eliminado | Integración con mensajería (RabbitMQ) no está implementada actualmente en el sistema. |
| CP-11 | Intento de transición inválida READY → IN_PREPARATION. | Sin ajuste | Regla de negocio válida que debe generar error 400. |
| CP-12 | Visualización del estado inicial del pedido (Pendiente). | Sin ajuste | Flujo funcional del cliente correctamente definido. |
| CP-13 | Actualización automática del estado mediante polling cada 5 segundos. | Ajustado: Polling ≤ 3 segundos | Se alinea con el comportamiento real esperado del sistema. |
| CP-14 | Visualización del estado final READY con indicador visual. | Sin ajuste | Caso funcional válido para el flujo final del pedido. |
| CP-15 | Consulta de pedido inexistente mediante UUID. | Sin ajuste | Manejo de error esperado (404 Not Found). |

---

# H1a: SELECCIÓN DE MESA (CLIENTE)

## Técnicas Aplicadas

- Análisis de valores límite
- Partición de equivalencia
- Pruebas negativas

---

### Caso de Prueba 1: Validación de número de mesa (límite inferior)

**Técnica:** Análisis de valores límite

**Dado que** el cliente se encuentra en la página `TableSelectPage.tsx`  
**Cuando** ingresa el número de mesa **0**  
**Entonces** el sistema debe mostrar un error de validación y no permitir avanzar al menú.

**Resultado esperado**

- Se muestra mensaje de error
- No hay navegación a `/client/menu`

---

### Caso de Prueba 2: Acceso exitoso con mesa válida

**Técnica:** Partición de equivalencia

**Dado que** el cliente se encuentra en la pantalla de selección de mesa  
**Cuando** ingresa el número de mesa **1**  
**Entonces** el sistema debe redirigir al cliente a `MenuPage.tsx`.

**Resultado esperado**

- Redirección correcta al menú
- Visualización de productos disponibles

---

# H1b: REALIZACIÓN DE PEDIDO (CLIENTE)

## Técnicas Aplicadas

- Pruebas de persistencia
- Validación funcional
- Pruebas negativas
- Integración API

---

### Caso de Prueba 3: Persistencia del carrito

**Técnica:** Prueba de persistencia

**Dado que** el cliente agrega productos al carrito  
**Cuando** refresca la página o cierra la pestaña del navegador  
**Entonces** el sistema debe recuperar los productos almacenados.

**Resultado esperado**

- Recuperación desde `localStorage`
- Cantidades y totales correctos

---

### Caso de Prueba 4: Confirmación exitosa del pedido

**Técnica:** Validación funcional

**Dado que** el cliente tiene un carrito válido  
**Cuando** presiona el botón **Confirmar Pedido**  
**Entonces** el sistema debe:

- Generar un **UUID único**
- Persistir la orden con estado **PENDING**
- Retornar **HTTP 201 Created**
- Redirigir a `ConfirmationPage.tsx`.

**Resultado esperado**

- Redirección a `/client/confirm/{orderId}`
- UUID visible en la interfaz
- Registro persistido en base de datos

---

# H2: GESTIÓN DE PEDIDOS (COCINA)

## Técnicas Aplicadas

- Transición de estados
- Validación de reglas de negocio
- Pruebas de seguridad

---

### Caso de Prueba 5: Visualización del tablero de cocina

**Técnica:** Prueba funcional

**Dado que** el personal de cocina accede al tablero  
**Cuando** existen pedidos en estado **PENDING**  
**Entonces** el sistema debe mostrarlos en el tablero ordenados por fecha de creación.

---

### Caso de Prueba 6: Transición válida de estado

**Técnica:** Máquina de estados

**Dado que** un pedido está en estado **PENDING**  
**Cuando** el cocinero presiona **Iniciar**  
**Entonces** el sistema debe cambiar el estado a **IN_PREPARATION**.

**Resultado esperado**

- Cambio de estado correcto
- Actualización del campo `updatedAt`

---

### Caso de Prueba 7: Transición inválida

**Técnica:** Prueba de robustez

**Dado que** un pedido está en estado **READY**  
**Cuando** se intenta cambiar a **IN_PREPARATION** mediante la API  
**Entonces** el sistema debe retornar **400 Bad Request**.

---

# H3: CONSULTA DE ESTADO DEL PEDIDO (CLIENTE)

## Técnicas Aplicadas

- Partición de equivalencia
- Pruebas de tiempo
- Pruebas negativas

---

### Caso de Prueba 8: Visualización del estado inicial

**Dado que** el cliente posee el UUID de su pedido  
**Cuando** accede a `/client/status/{orderId}`  
**Entonces** el sistema debe mostrar el estado **PENDING**.

**Resultado esperado**

- Badge amarillo
- Detalles del pedido visibles

---

### Caso de Prueba 9: Actualización automática del estado

**Técnica:** Pruebas de sincronización

**Dado que** el cliente visualiza la pantalla de estado  
**Cuando** la cocina cambia el estado del pedido  
**Entonces** el sistema debe actualizar la vista mediante polling.

**Resultado esperado**

- Actualización automática ≤ 3 segundos
- Cambio visual del estado

---

### Caso de Prueba 10: Pedido listo

**Dado que** el pedido está en estado **READY**  
**Cuando** el cliente consulta su estado  
**Entonces** el sistema debe mostrar el estado final.

**Resultado esperado**

- Badge verde
- Mensaje indicando que el pedido está listo

---

### Caso de Prueba 11: Pedido inexistente

**Técnica:** Prueba negativa

**Dado que** el cliente ingresa un UUID inválido  
**Cuando** el frontend realiza `GET /orders/{id}`  
**Entonces** el sistema debe retornar **404 Not Found**.

**Resultado esperado**

- Mensaje indicando que el pedido no fue encontrado