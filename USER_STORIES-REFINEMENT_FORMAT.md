# Refinamiento de Historias de Usuario

Documento que compara las Historias de Usuario (HU) originales con las versiones refinadas generadas por la gema, identificando las diferencias relevantes para la implementación del proyecto **"Sistema de pedidos restaurante"**.

---

# H1 — Pedido desde el menú digital

## Historia Original

El cliente del restaurante quiere seleccionar la mesa y crear un pedido desde el menú digital para que la cocina lo reciba y lo prepare sin intermediarios.

---

## Historia Refinada

### H1a — Selección de Mesa (Cliente)

**Título:** Identificación de mesa en el menú digital.

**Descripción:**  
Como cliente del restaurante, quiero seleccionar mi mesa en el menú digital para asegurar que mi pedido sea entregado en el lugar correcto.

**Criterios de aceptación**

- **Criterio 1**  
  Dado que estoy en la pantalla de inicio del menú,  
  cuando selecciono la mesa desde el menú,  
  entonces el sistema debe permitirme ver la carta de productos.

- **Criterio 2**  
  Dado que ya tengo una mesa asignada,  
  cuando intento ingresar un número de mesa diferente durante la misma sesión,  
  entonces el sistema debe mantenerme en la mesa original.

---

### H1b — Realización de Pedido (Cliente)

**Título:** Envío del pedido directamente a cocina.

**Descripción:**  
Como cliente del restaurante, quiero enviar mi pedido directamente a la cocina para recibir mi comida sin intermediarios.

**Criterios de aceptación**

- **Criterio 1**  
  Dado que estoy navegando en el menú,  
  cuando consulto los platos disponibles,  
  entonces solo debo ver aquellos que están activos.

- **Criterio 2**  
  Dado que tengo platos en mi carrito,  
  cuando uno de ellos deja de estar disponible antes de confirmar,  
  entonces el sistema debe avisarme y evitar que finalice el pedido con ese producto.

- **Criterio 3**  
  Dado que cierro el navegador por error,  
  cuando vuelvo a abrir el menú en la misma sesión,  
  entonces mis productos seleccionados deben seguir en el carrito.

- **Criterio 4**  
  Dado que confirmo mi pedido,  
  cuando el envío es exitoso,  
  entonces debo recibir una confirmación de que la cocina ya lo está preparando.

---

## Diferencias detectadas

- La historia original se **divide en dos historias más pequeñas**.
- Se agregan **criterios de aceptación en formato Gherkin**.
- Se definen **reglas de negocio** como:
  - mantener la mesa durante la sesión
  - mostrar solo platos disponibles
  - validar disponibilidad antes de confirmar.
- Las historias refinadas son **más fáciles de probar y automatizar con BDD**.

---

# H2 — Gestión de pedidos en cocina

## Historia Original

El personal de cocina quiere visualizar los pedidos pendientes y cambiar su estado para gestionar su preparación de forma organizada.

---

## Historia Refinada

**Historia de Usuario:** Gestión de pedidos en cocina

**Descripción:**  
Como cocinero, requiero visualizar los pedidos entrantes y actualizar su progreso para organizar la preparación de los platos y asegurar que se entreguen a tiempo.

**Criterios de aceptación**

- **Visualización de pendientes**  
  Dado que hay pedidos confirmados por los clientes,  
  cuando accedo al monitor de cocina,  
  entonces debo ver una lista de pedidos pendientes ordenada por tiempo de llegada.

- **Cambio de estado a "En preparación"**  
  Dado que selecciono un pedido,  
  cuando lo marco como "En preparación",  
  entonces el sistema debe actualizar su estado para que otros cocineros sepan que ya se está atendiendo.

- **Finalización del pedido**  
  Dado que termino de preparar todos los platos de un pedido,  
  cuando lo marco como "Listo",  
  entonces el pedido debe desaparecer de la lista de pendientes y notificarse al personal de salón o al cliente.

---

## Diferencias detectadas

- La refinada **explica mejor el objetivo del sistema**.
- Se agregan **criterios de aceptación en formato BDD**.
- Se define el **flujo de estados del pedido**:
  - Pendiente
  - En preparación
  - Listo.
- Se especifica **cómo debe reaccionar el sistema cuando cambia el estado**.

---

# H3 — Consulta de estado del pedido

## Historia Original

El cliente del restaurante quiere consultar el estado del pedido en tiempo real para saber cuándo está listo para servir.

---

## Historia Refinada

**Historia de Usuario:** Consulta de estado del pedido en tiempo real

**Descripción:**  
Como cliente del restaurante, quiero consultar el estado de mi pedido en tiempo real para saber cuándo está listo para ser servido y reducir la incertidumbre durante la espera.

**Criterios de aceptación**

- **Visualización del estado inicial**  
  Dado que he confirmado mi pedido,  
  cuando ingreso a la sección "Mis pedidos",  
  entonces debo visualizar el estado actual del pedido.

- **Actualización de progreso**  
  Dado que mi pedido está siendo preparado,  
  cuando consulto la pantalla,  
  entonces el estado debe mostrarse como "En preparación".

- **Notificación de finalización**  
  Dado que el pedido ha sido marcado como "Listo" por cocina,  
  cuando reviso mi dispositivo,  
  entonces debo ver un aviso indicando que el pedido está listo.

---

## Diferencias detectadas

- La refinada **explica mejor la necesidad del usuario** (reducir la incertidumbre de espera).
- Se agregan **criterios de aceptación en formato Gherkin/BDD**.
- Se define el **flujo de estados visible para el cliente**.
- Se especifica **cuándo debe actualizarse la información del pedido**.