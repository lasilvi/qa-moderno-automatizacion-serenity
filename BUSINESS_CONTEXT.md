# CONTEXTO DE NEGOCIO
## Sistema de Pedidos de Restaurante

**Versión:** 1.0  
**Fecha de Análisis:** 2 de marzo de 2026  
**Analista:** Arquitecto de Software Senior  
**Fuente:** Análisis completo del código fuente en repositorios backend y frontend

---

## 1. DESCRIPCIÓN DEL PROYECTO

### Nombre del Proyecto
**Sistema de Pedidos de Restaurante** (Restaurant Order Management System)

### Objetivo del Proyecto

Desarrollar una plataforma digital integral para la gestión de pedidos en restaurantes que automatice el flujo completo desde la toma del pedido por parte del cliente hasta su preparación en cocina y generación de reportes operacionales.

---

## 2. FLUJOS CRÍTICOS DEL NEGOCIO

### Principales Flujos de Trabajo

#### **FLUJO 1: Creación de Pedido (Cliente)**

**Actor:** Cliente del restaurante  
**Trigger:** Ingreso a la aplicación web desde dispositivo móvil/desktop  
**Evidencia en código:** `MenuPage.tsx`, `CartPage.tsx`, `OrderController.java`

**Pasos del flujo:**

1. **Selección de mesa** (`TableSelectPage.tsx`)
   - El cliente ingresa el número de mesa (validación: entero positivo >= 1)
   - El número de mesa se almacena en contexto global de React (`useApp()`)
   - **Restricción identificada:** No se valida límite superior de mesas (ausencia de `@Max(12)` documentada en docs/week-3-review/REST_API_AUDIT.md)

2. **Navegación del menú** (`MenuPage.tsx` + `GET /menu`)
   - Sistema solicita todos los productos activos al backend (`ProductRepository.findByIsActiveTrue()`)
   - Frontend organiza productos por categorías: `entradas`, `principales`, `postres`, `bebidas`
   - Filtrado en tiempo real mediante barra de búsqueda (búsqueda textual sobre `name` y `description`)

3. **Adición al carrito** (componente de estado local en React)
   - Cliente especifica cantidad (validación: >= 1, máximo 99 según `<Input type="number" min={1} max={99}/>`)
   - Opcionalmente agrega notas especiales por item (`note?: string`)
   - Estado del carrito persistido en `CartContext` durante la sesión

4. **Revisión y confirmación** (`CartPage.tsx`)
   - Resumen visual: items, cantidades, subtotales, total general
   - Botón "Confirmar Pedido" invoca `POST /orders` con payload:
     ```json
     {
       "tableId": 5,
       "items": [
         { "productId": 1, "quantity": 2, "note": "Sin cebolla" }
       ]
     }
     ```

5. **Persistencia backend** (`OrderService.createOrder()`)
   - **Validaciones ejecutadas** (clase `OrderValidator`):
     - Todos los `productId` existen en base de datos
     - Todos los productos referenciados tienen `isActive = true`
     - `tableId` es >= 1
     - Lista de items no está vacía
   - Creación de entidad `Order` con estado inicial `PENDING`
   - Generación automática de UUID como identificador único
   - Timestamps `createdAt` y `updatedAt` establecidos por `@PrePersist`
   - Soft delete flags: `deleted = false`, `deletedAt = null`

6. **Publicación de evento** (`RabbitOrderPlacedEventPublisher`)
   - Construcción de `OrderPlacedDomainEvent` con versión de contrato (`eventVersion: 2`)
   - Publicación a exchange `order.exchange` con routing key `order.placed`
   - Patrón Command: `PublishOrderPlacedEventCommand` ejecutado por `OrderCommandExecutor`

7. **Confirmación al cliente** (`ConfirmationPage.tsx`)
   - Redirección a `/client/confirm/{orderId}` con UUID del pedido
   - Presentación de resumen y botón para seguimiento

**Regla de negocio crítica:** Si algún producto en el carrito se desactiva entre la visualización del menú y la confirmación, el backend rechaza la orden completa con `400 Bad Request` y mensaje "Product with ID X is not active".

---

#### **FLUJO 2: Procesamiento Asíncrono en Cocina**

**Actor:** Kitchen Worker Service (microservicio sin interfaz humana)  
**Trigger:** Recepción de evento `order.placed` desde RabbitMQ  
**Evidencia en código:** `OrderEventListener.java`, `OrderProcessingService.java`

**Pasos del flujo:**

1. **Consumo del evento** (`@RabbitListener`)
   - Queue configurada: `order.placed.queue`
   - Binding al exchange `order.exchange`
   - Dead Letter Exchange configurado: `order.dlx` → `order.placed.dlq`

2. **Validación de contrato** (`OrderPlacedEventValidator`)
   - Verificación de estructura JSON según esquema esperado
   - Validación de versión del evento (soporta v1 y v2 con retrocompatibilidad)
   - **Si falla validación:** Mensaje enviado inmediatamente a DLQ sin reintentos (error de contrato no recuperable)

3. **Verificación de orderId** (`OrderProcessingService.processOrder()`)
   - Búsqueda en `kitchen_orders` por UUID
   - **Si no existe:** Crea proyección local con datos del evento (`tableId`, `orderId`, `status = PENDING`, `createdAt`)
   - **Si ya existe:** Actualiza registro existente

4. **Cambio de estado a IN_PREPARATION**
   - Update SQL: `status = 'IN_PREPARATION'`, `updatedAt = NOW()`
   - Transacción garantizada con `@Transactional`

5. **Logging y observabilidad**
   - Log estructurado: `orderId`, `tableId`, `newStatus`
   - **Nota:** No se encontró evidencia de métricas Prometheus/Micrometer expuestas

6. **Manejo de errores transitorios**
   - Excepciones no capturadas → retry exponencial configurado en RabbitMQ
   - Máximo 3 reintentos con backoff
   - Tras 3 fallos → mensaje enviado a DLQ

**Proyección de datos:** `kitchen_db` mantiene una copia simplificada de pedidos (solo `orderId`, `tableId`, `status`, timestamps) sin items ni detalles de productos.

---

#### **FLUJO 3: Administración de Pedidos (Cocina)**

**Actor:** Personal de cocina autenticado  
**Trigger:** Login con token estático en `/kitchen`  
**Evidencia en código:** `KitchenBoardPage.tsx`, `KitchenSecurityInterceptor.java`

**Pasos del flujo:**

1. **Autenticación** (`KitchenLoginPage.tsx`)
   - Usuario ingresa token de cocina (valor esperado configurado en `.env`: `KITCHEN_AUTH_TOKEN=cocina123`)
   - Token se almacena en `localStorage` del navegador
   - **Mecanismo de seguridad:** Header HTTP `X-Kitchen-Token` en cada request

2. **Validación backend** (Chain of Responsibility)
   - `KitchenEndpointScopeHandler`: Verifica que el endpoint requiere autenticación (paths con `/orders/{id}/status`)
   - `KitchenTokenPresenceHandler`: Valida presencia del header `X-Kitchen-Token`
   - `KitchenTokenValueHandler`: Compara valor del header con `${security.kitchen.token-value}` en configuración
   - **Si falla:** Lanza `KitchenAccessDeniedException` → `401 Unauthorized`

3. **Visualización del tablero** (`GET /orders?status=PENDING,IN_PREPARATION`)
   - Polling cada 3 segundos (`POLL_MS = 3000`)
   - Tres columnas kanban: `PENDING`, `IN_PREPARATION`, `READY`
   - Cada tarjeta muestra: número de mesa, items del pedido, tiempo transcurrido desde creación

4. **Transición de estado** (`PATCH /orders/{id}/status`)
   - Botones: "Iniciar" (PENDING → IN_PREPARATION), "Marcar Listo" (IN_PREPARATION → READY)
   - **Validación de transición:** Enum `OrderStatus` con mapa de transiciones válidas:
     - `PENDING` → solo puede pasar a `IN_PREPARATION`
     - `IN_PREPARATION` → solo puede pasar a `READY`
     - `READY` → estado final, no permite más cambios
   - **Si transición inválida:** Backend retorna `400 Bad Request` con mensaje "Invalid status transition from X to Y"

5. **Publicación de evento `order.ready`** (solo al alcanzar estado `READY`)
   - Evento enviado a report-service para actualizar estadísticas
   - Routing key: `order.ready`

6. **Eliminación lógica** (`DELETE /orders/{id}`)
   - Soft delete: actualiza `deleted = true`, `deletedAt = NOW()`
   - **Ausencia de protección:** No requiere confirmación adicional (riesgo documentado en auditoría)
   - Pedido eliminado desaparece de todas las vistas pero permanece en base de datos para auditoría

---

#### **FLUJO 4: Consulta de Estado (Cliente)**

**Actor:** Cliente del restaurante  
**Trigger:** Navegación a `/client/status/{orderId}` desde confirmación o URL directa  
**Evidencia en código:** `OrderStatusPage.tsx`, `GET /orders/{id}`

**Pasos del flujo:**

1. **Solicitud de estado** (`GET /orders/{id}`)
   - Request con UUID obtenido tras creación de pedido
   - **No requiere autenticación** (endpoint público)

2. **Respuesta del backend**
   ```json
   {
     "id": "550e8400-e29b-41d4-a716-446655440000",
     "tableId": 5,
     "status": "IN_PREPARATION",
     "items": [
       { "id": 1, "productId": 1, "quantity": 2, "note": "Sin cebolla" }
     ],
     "createdAt": "2024-01-15T10:30:00",
     "updatedAt": "2024-01-15T10:35:12"
   }
   ```

3. **Renderizado en frontend**
   - Badge visual según estado:
     - `PENDING`: amarillo, texto "Pendiente"
     - `IN_PREPARATION`: azul, texto "En preparación"
     - `READY`: verde, texto "Listo para servir"
   - Lista de items con nombres de productos resueltos desde cache del menú
   - Tiempo estimado: "Hace X minutos" calculado desde `updatedAt`

4. **Actualización en tiempo real**
   - Polling cada 5 segundos si estado no es `READY`
   - Cuando alcanza `READY`, se detiene el polling y muestra notificación

---

#### **FLUJO 5: Generación de Reportes**

**Actor:** Personal administrativo (gerente, contador)  
**Trigger:** Navegación a `/reports/orders` con selección de rango de fechas  
**Evidencia en código:** `OrdersReportPage.tsx`, `ReportController.java`

**Pasos del flujo:**

1. **Construcción de proyección CQRS** (automática, basada en eventos)
   - `report-service` consume eventos `order.placed` y `order.ready`
   - Tabla desnormalizada `report_orders` incluye:
     - `orderId`, `tableId`, `status`, `createdAt`, `receivedAt`
     - Items en tabla relacionada `report_order_items` con `productId`, `quantity`, `price` (snapshot del precio al momento de la orden)

2. **Consulta de reporte** (`GET /reports?startDate=2024-01-01&endDate=2024-01-31`)
   - Filtrado por rango de fechas aplicado sobre `createdAt`
   - Solo se consideran órdenes con estado `READY` (completas)

3. **Agregaciones en dominio** (`ReportAggregationService`)
   - Cálculo en memoria Java (no en SQL) para fácil testing:
     - `totalReadyOrders`: conteo de órdenes en estado `READY`
     - `totalRevenue`: suma de `price * quantity` de todos los items
     - `productBreakdown`: mapa agrupado por `productId` con suma de cantidades vendidas

4. **Respuesta estructurada**
   ```json
   {
     "totalReadyOrders": 42,
     "totalRevenue": 125750,
     "productBreakdown": [
       {
         "productId": 1,
         "productName": "Empanadas criollas",
         "quantitySold": 85,
         "totalAccumulated": 38250
       }
     ]
   }
   ```

5. **Visualización en frontend**
   - Cards con métricas principales (total órdenes, ingresos totales)
   - Tabla de productos con ranking por cantidad vendida
   - Gráfico de barras con top 10 productos (implementación con librería de gráficos no encontrada en código actual, solo estructura de datos)

**Nota crítica:** La proyección CQRS está eventualmente consistente. Si un evento falla al procesarse, el reporte mostrará datos desactualizados hasta que el mensaje se reintente desde la DLQ o se reprocese manualmente.

---

### Módulos o Funcionalidades Críticas

| Módulo | Servicio | Criticidad | Justificación |
|--------|----------|------------|---------------|
| **Gestión de pedidos** | `order-service` | **CRÍTICA** | Punto de entrada único para creación y consulta de pedidos. Su caída bloquea operativa completa del negocio. |
| **Publicación de eventos** | `order-service` | **CRÍTICA** | Si falla RabbitMQ o el publisher, los pedidos se crean en BD pero la cocina nunca los recibe. Genera inconsistencia operacional. |
| **Procesamiento asíncrono** | `kitchen-worker` | **ALTA** | Su caída impide que pedidos nuevos pasen a estado `IN_PREPARATION`. Pedidos existentes en cocina pueden completarse manualmente. |
| **Autenticación de cocina** | `order-service` (security module) | **ALTA** | Previene modificaciones no autorizadas de estados. Fallo permitiría transiciones inválidas o eliminación masiva de pedidos. |
| **Validación de productos** | `order-service` (OrderValidator) | **ALTA** | Evita pedidos de items inexistentes o inactivos. Su ausencia causaría errores en cocina al no poder resolver nombres de productos. |
| **Reportería** | `report-service` | **MEDIA** | No impacta operativa del día a día. Importante para análisis de negocio pero tolera desfase temporal. |
| **Soft delete** | `order-service` | **MEDIA** | Garantiza trazabilidad para auditorías. Su fallo no impide operación pero compromete compliance regulatorio. |

---

## 3. REGLAS DE NEGOCIO Y RESTRICCIONES

### Reglas de Negocio Relevantes

#### **RN-001: Máquina de Estados de Pedido**

**Definición:** Los pedidos siguen un flujo unidireccional estricto de estados que no puede revertirse.

**Flujo permitido:**
```
PENDING → IN_PREPARATION → READY
```

**Implementación:** Enum `OrderStatus` con mapa de transiciones válidas en método estático `validateTransition()`.

**Excepción lanzada:** `InvalidStatusTransitionException` si se intenta transición no permitida.

**Evidencia en código:**
```java
private static final Map<OrderStatus, Set<OrderStatus>> VALID_TRANSITIONS = Map.of(
    PENDING, EnumSet.of(IN_PREPARATION),
    IN_PREPARATION, EnumSet.of(READY),
    READY, EnumSet.noneOf(OrderStatus.class)
);
```

**Rationale de negocio:** Una vez marcado como listo, un pedido no puede volver a estados anteriores para evitar confusión operacional (e.g., marcar listo por error y querer revertir a "en preparación" causaría que el cliente ya haya recibido notificación).

**Restricción técnica:** No existe un estado `CANCELLED`. Si se requiere cancelar un pedido, debe eliminarse lógicamente con soft delete.

---

#### **RN-002: Validación de Productos Activos**

**Definición:** Solo pueden agregarse a un pedido productos que existan en el catálogo y estén marcados como activos (`isActive = true`).

**Implementación:** Clase `OrderValidator.validateCreateOrderRequest()` ejecuta queries a `ProductRepository.findById()` para cada `productId` en el request.

**Excepciones lanzadas:**
- `ProductNotFoundException`: Si el `productId` no existe en tabla `products`
- `InactiveProductException`: Si el producto existe pero `isActive = false`

**Evidencia documental:** Documentado como requerimientos 2.2 y 2.6 en comentarios Javadoc de `OrderService.createOrder()`.

**Caso límite identificado:** Si un producto se desactiva entre la carga del menú en frontend y la confirmación del pedido (ventana típica de 2-5 minutos), el backend rechaza la orden completa. El frontend no tiene mecanismo de reintento parcial eliminando solo el ítem inactivo.

---

#### **RN-003: Integridad Referencial de Items**

**Definición:** Cada `OrderItem` mantiene solo el `productId` (ID numérico), no una copia completa del producto. El nombre y precio del producto deben resolverse mediante join o cache.

**Implementación:**
- En `order-service`: La entidad `OrderItem` tiene campo `productId` de tipo `Long` sin relación JPA bidireccional.
- En `report-service`: El evento `OrderPlacedEvent` incluye snapshot de items con sus precios al momento de la orden para evitar dependency en el catálogo de productos.

**Implicación:** Si un producto es editado (cambio de nombre o precio) después de creado el pedido, la orden sigue referenciando el ID original pero el nombre mostrado en UI puede cambiar. Para reportería histórica, el `report-service` captura el precio en el momento de la venta.

---

#### **RN-004: Soft Delete Obligatorio**

**Definición:** Los pedidos nunca se eliminan físicamente de la base de datos. La operación `DELETE /orders/{id}` establece flags `deleted = true` y `deleted_at = NOW()`.

**Implementación:** 
- Método `OrderService.deleteOrder()` ejecuta `order.setDeleted(true)` y persiste con `save()`.
- Query method personalizado: `OrderRepository.findByIdAndDeletedFalse()` filtra pedidos eliminados en todas las lecturas.

**Rationale de negocio:** Cumplimiento con requerimientos de auditoría y trazabilidad regulatoria. Los pedidos eliminados pueden necesitarse para:
- Disputas de clientes
- Auditorías contables
- Análisis retrospectivo de errores operacionales

**Evidencia documental:** Documentado explícitamente en comentarios de la entidad `Order` con referencia a "Copilot Instructions Section 4".

**Limitación identificada:** No existe interfaz de administración para recuperar pedidos soft-deleted. Solo son accesibles vía SQL directo o eliminando el filtro `deletedFalse` en queries.

---

#### **RN-005: Versionado de Eventos**

**Definición:** Los eventos AMQP incluyen campo `eventVersion` para soportar evolución del contrato sin romper consumidores legacy.

**Versiones detectadas:**
- **v1 (legacy):** Campos `id` y `table_id` (snake_case)
- **v2 (actual):** Campos `orderId` y `tableId` (camelCase) + campo adicional `createdAt`

**Implementación combatibilidad:**
```java
public UUID resolveOrderId() {
    return orderId != null ? orderId : id; // Fallback a v1
}
```

**Rationale técnico:** Permite despliegue independiente de servicios. Un consumidor puede actualizar su lógica para leer v2 mientras otros aún parsean v1, evitando "big bang deployment".

**Restricción actual:** El validador de `kitchen-worker` rechaza versiones desconocidas (>2) enviándolas a DLQ. No hay estrategia de forward compatibility.

---

### Regulaciones o Normativas Aplicables

#### **Protección de Datos Personales**

**Estatus:** **NO APLICA**

**Análisis:** El sistema no recopila datos personales identificables (PII):
- No hay registro de usuarios (ni clientes ni staff de cocina)
- No se capturan nombres, emails, teléfonos o direcciones
- El identificador único es el número de mesa (entity sin datos personales)

**Consideraciones futuras:** Si se añade módulo de fidelización o pagos online, aplicaría:
- **Colombia:** Ley 1581 de 2012 (Protección de Datos Personales)
- **GDPR** (si opera en UE): Consentimiento explícito, derecho al olvido

---

#### **Facturación Electrónica**

**Estatus:** **NO IMPLEMENTADO**

**Análisis:** El sistema no genera facturas ni documentos tributarios:
- No hay integración con DIAN (Colombia)
- No se registra información fiscal de clientes
- No se calculan impuestos (IVA, propinas)

**Implicación:** El sistema funciona como herramienta de gestión operativa pre-fiscal. La facturación debe hacerse en sistema externo (POS fiscal) tomando datos del campo `totalRevenue` del reporte.

---

#### **Trazabilidad Alimentaria**

**Estatus:** **PARCIALMENTE IMPLEMENTADO**

**Análisis:**
- ✅ Timestamps de creación y actualización en cada pedido
- ✅ Registro inmutable con soft delete (auditoría completa)
- ❌ No se registra staff que preparó el pedido
- ❌ No se registra batch/lote de ingredientes usados
- ❌ No se registra staff que sirvió el pedido

**Regulación aplicable (si maneja alimentos perecederos):**
- **FDA Food Safety Modernization Act** (FSMA) si exporta a EEUU
- **NOM-251-SSA1-2009** (México) sobre prácticas de higiene

**Recomendación:** Si el restaurante requiere trazabilidad end-to-end, agregar campos `preparedBy` y `servedBy` en entidad `Order` con timestamp separado para cada transición de estado.

---

#### **Accesibilidad Web**

**Estatus:** **CUMPLIMIENTO PARCIAL**

**Análisis del frontend:**
- ✅ Uso de semantic HTML (`<button>`, `<nav>`, `<main>`)
- ✅ Contraste de colores gestionado por Tailwind (tema claro/oscuro)
- ⚠️ No se encontraron atributos ARIA en componentes custom
- ⚠️ No se encontró navegación por teclado explícita en `KitchenBoardPage`

**Normativas aplicables:**
- **WCAG 2.1 Level AA** (estándar internacional)
- **Section 508** (EEUU) si presta servicios a entidades gubernamentales

**Recomendación:** Auditoría con herramientas automatizadas (axe DevTools, WAVE) y pruebas con lectores de pantalla (NVDA, JAWS).

---

## 4. PERFILES DE USUARIO Y ROLES

### Perfiles o Roles de Usuario

El sistema identifica **dos perfiles principales** basados en análisis del código fuente:

#### **PERFIL 1: Cliente del Restaurante**

**Características demográficas inferidas:**
- Usuario final sin conocimiento técnico esperado
- Acceso desde dispositivos móviles (diseño responsivo detectado en `tailwind.config.cjs`)
- Expectativa de UX simplificada tipo e-commerce

**Capacidades en el sistema:**
- Seleccionar número de mesa
- Navegar catálogo de productos (filtrado por categoría y búsqueda textual)
- Agregar items al carrito con cantidades y notas
- Confirmar pedido (crea transacción en backend)
- Consultar estado de pedido en tiempo real

**Restricciones:**
- No puede modificar estado de pedidos
- No puede eliminar pedidos una vez confirmados
- No puede acceder a estadísticas o reportes
- No requiere autenticación formal (identificación implícita por número de mesa)

**Rutas frontend asignadas:**
- `/client/table` - Selección de mesa
- `/client/menu` - Catálogo de productos
- `/client/cart` - Revisión de carrito
- `/client/confirm/:orderId` - Confirmación
- `/client/status/:orderId` - Seguimiento

**Evidencia en código:** 
```tsx
// src/App.tsx
<Route path="/client/table" element={<TableSelectPage />} />
<Route path="/client/menu" element={<MenuPage />} />
```

---

#### **PERFIL 2: Personal de Cocina**

**Características operacionales:**
- Usuario interno del restaurante
- Acceso desde terminal fija en área de cocina (no móvil)
- Requiere velocidad de visualización (polling cada 3 segundos)

**Capacidades en el sistema:**
- Visualizar todos los pedidos activos (kanban con 3 columnas)
- Cambiar estado de pedidos según flujo permitido
- Marcar pedidos como listos para servir
- Eliminar pedidos erróneos (soft delete)
- Limpiar historial completo de pedidos (operación destructiva)

**Restricciones:**
- No puede crear pedidos (ese flujo es exclusivo de cliente)
- No puede editar items de un pedido (solo cambiar estado completo)
- No puede acceder a reportes de ingresos (separación de responsabilidades)

**Rutas frontend asignadas:**
- `/kitchen` - Login con token
- `/kitchen/board` - Tablero kanban (requiere autenticación)

**Mecanismo de autenticación:**
- Token estático compartido entre todo el staff de cocina
- Valor configurado en variable de entorno: `KITCHEN_AUTH_TOKEN=cocina123`
- Transmisión vía header HTTP: `X-Kitchen-Token`
- Validación en backend mediante interceptor `KitchenSecurityInterceptor`

**Evidencia en código:**
```tsx
// src/pages/kitchen/KitchenBoardPage.tsx - Requiere token
const token = getKitchenToken()

// src/store/kitchenAuth.ts
export function getKitchenToken(): string | null {
  return localStorage.getItem(STORAGE_KEY)
}
```

```java
// KitchenSecurityInterceptor.java
public KitchenSecurityInterceptor(
    @Value("${security.kitchen.token-header}") String tokenHeaderName,
    @Value("${security.kitchen.token-value}") String expectedToken
)
```

---

### Permisos y Limitaciones de cada Perfil

| Operación | Cliente | Cocina |
|-----------|---------|--------|
| **Ver menú** | ✅ | ✅ (indirectamente) |
| **Crear pedido** | ✅ | ❌ | ❌ |
| **Ver pedido específico** | ✅ (solo propio) | ✅ (todos) |
| **Cambiar estado pedido** | ❌ | ✅ (con restricciones de flujo) |
| **Eliminar pedido** | ❌ | ✅ (soft delete) |
| **Ver reportes** | ❌ | ❌ |
| **Administrar productos** | ❌ | ❌ |
| **Administrar mesas** | ❌ | ❌ |

**Leyenda:**
- ✅ Permitido explícitamente
- ❌ Bloqueado por diseño
- ⚠️ No implementado o mal protegido

---

## 5. CONDICIONES DEL ENTORNO TÉCNICO

### Plataformas Soportadas

#### **Frontend (React SPA)**

**Framework y versiones:**
- React 18.3.1
- Vite 5.4.2 (build tool)
- TypeScript 5.5.4

**Compatibilidad de navegadores:**
- **No especificada explícitamente** en `package.json`
- Inferida de dependencias modernas: requiere browsers con soporte `ES2020`
- Compatibles: Chrome 90+, Firefox 88+, Safari 14+, Edge 90+
- **No soporta:** Internet Explorer (descontinuado)

**Dispositivos objetivo:**
- Desktop (responsive grid con Tailwind)
- Tablets (layouts adaptativos confirmados en uso de `@media`)
- Smartphones (menú y carrito optimizados para touch)

**Evidencia:** Configuración Tailwind con breakpoints estándar:
```js
// tailwind.config.cjs
theme: {
  extend: {
    screens: {
      'sm': '640px',
      'md': '768px',
      'lg': '1024px',
      'xl': '1280px'
    }
  }
}
```

**Accesibilidad:** Tema claro/oscuro con toggle manual (`ThemeToggle.tsx`), mejora legibilidad en ambientes con iluminación variable (cocinas con luz natural vs. áreas de servicio con luz artificial).

---

#### **Backend (Spring Boot Microservices)**

**Framework principal:**
- Spring Boot 3.2.0
- Java 17 (LTS)
- Maven 3.9+ (inferido de `pom.xml` syntax)

**Plataformas de despliegue:**
- Docker (Dockerfiles para cada servicio)
- Kubernetes-ready (ausencia de dependencies específicas de VM)
- Cloud-agnostic (no se detectaron dependencias de AWS/Azure/GCP)

**Contenedores en producción:**
```yaml
# docker-compose.yml
order-service:     puerto 8080
kitchen-worker:    sin puerto expuesto (worker asíncrono)
report-service:    puerto 8082
```

**Bases de datos:**
- PostgreSQL 15
- 3 instancias separadas (database-per-service pattern)
  - `restaurant_db` - puerto 5432
  - `kitchen_db` - puerto 5433
  - `report_db` - puerto 5434

**Message broker:**
- RabbitMQ 3.x con plugin de management
- Puerto AMQP: 5672
- Puerto UI admin: 15672

---

### Tecnologías o Integraciones Clave

#### **Stack Tecnológico Completo**

##### **Backend - Principales Dependencias**

| Tecnología | Versión | Propósito | Módulo |
|------------|---------|-----------|--------|
| Spring Boot Web | 3.2.0 | REST API controllers | order-service, report-service |
| Spring Data JPA | 3.2.0 | ORM con PostgreSQL | Todos los servicios |
| Spring AMQP | 3.2.0 | Messaging con RabbitMQ | Todos los servicios |
| PostgreSQL Driver | 42.7.1 | Conexión a BD | Todos los servicios |
| Flyway | 10.x | Migraciones de BD | Todos los servicios |
| Lombok | 1.18.30 | Reducción de boilerplate | Todos los servicios |
| SpringDoc OpenAPI | 2.x | Documentación Swagger | order-service, report-service |
| JUnit 5 | 5.10+ | Testing unitario | Todos los servicios |
| jqwik | 1.7.4 | Property-based testing | order-service |
| JaCoCo | 0.8.11 | Cobertura de código | Todos los servicios |

**Evidencia:**
```xml
<!-- pom.xml -->
<spring-boot.version>3.2.0</spring-boot.version>
<postgresql.version>42.7.1</postgresql.version>
<jqwik.version>1.7.4</jqwik.version>
```

---

##### **Frontend - Principales Dependencias**

| Tecnología | Versión | Propósito |
|------------|---------|-----------|
| React | 18.3.1 | UI library |
| React Router | 6.26.2 | Client-side routing |
| TanStack Query | 5.56.2 | Server state management |
| Tailwind CSS | 3.4.10 | Utility-first styling |
| Lucide React | 0.468.0 | Iconografía (fork de Feather Icons) |
| Motion | 12.23.24 | Animaciones (fork de Framer Motion) |
| Vitest | 4.x | Testing framework |
| Testing Library | 16.3.2 | DOM testing utilities |

**Evidencia:**
```json
// package.json
"dependencies": {
  "@tanstack/react-query": "^5.56.2",
  "react": "^18.3.1",
  "motion": "^12.23.24"
}
```

---

#### **Integraciones Externas**

##### **Imágenes de Productos (Unsplash CDN)**

**Propósito:** Proporcionar imágenes de alta calidad para items del menú sin almacenamiento local.

**Implementación:**
- URLs almacenadas en campo `image_url` de tabla `products`
- Ejemplos detectados en datos semilla:
  ```
  https://images.unsplash.com/photo-1603360946369-dc9bb6258143
  https://images.unsplash.com/photo-1558030006-450675393462
  ```

**Dependencia:** 
- **Tipo:** Soft dependency
- **Impacto de caída:** Imágenes no se cargan pero la funcionalidad continúa (frontend muestra placeholder)

**Evidencia en código:**
```tsx
// ProductImage.tsx
<img 
  src={product.imageUrl} 
  onError={(e) => { e.currentTarget.src = '/placeholder.svg' }}
/>
```

**Consideraciones:**
- Sin SLA garantizado (servicio gratuito de terceros)
- Potencial pérdida de imágenes si URLs cambian
- Recomendación: Migrar a CDN propio (Cloudflare Images, AWS S3) para entorno productivo

---

##### **GitHub Container Registry (GHCR)**

**Propósito:** Almacenamiento y distribución de imágenes Docker para CI/CD.

**Configuración detectada:**
```yaml
# docker-compose.yml
order-service:
  image: ghcr.io/maese-alfred/sistemas-de-pedidos-restaurante/order-service:latest

kitchen-worker:
  image: ghcr.io/maese-alfred/sistemas-de-pedidos-restaurante/kitchen-worker:latest
```

**Pipeline identificado:**
- CI en GitHub Actions (archivo `.github/workflows/ci-cd.yml` listado en estructura)
- Build multi-stage para optimizar tamaño de imágenes
- Push automático a GHCR en cada commit a rama `main`

**Dependencia:**
- **Tipo:** Hard dependency para deployment automatizado
- **Impacto de caída:** Imposible desplegar nuevas versiones, pero instancias corriendo no se ven afectadas

---

##### **SonarCloud (Análisis Estático)**

**Propósito:** Quality gate automático en pipeline CI/CD.

**Configuración:**
```xml
<!-- pom.xml -->
<sonar.organization>Maese-Alfred</sonar.organization>
<sonar.host.url>https://sonarcloud.io</sonar.host.url>
<sonar.projectKey>Maese-Alfred_Sistemas-de-pedidos-restaurante-backend</sonar.projectKey>
```

**Métricas rastreadas:**
- Cobertura de código (target: >80% según estándar de proyecto)
- Code smells y technical debt
- Vulnerabilidades de seguridad (Sonar Security Hotspots)

**Dependencia:**
- **Tipo:** Soft dependency (desarrollo)
- **Impacto de caída:** Pipeline puede fallar pero no afecta runtime

---

#### **Diagrama de Integración**

```
┌─────────────────────────────────────────────────────────────────┐
│                        FRONTEND (React)                         │
│                       Puerto 5173 / 8080                        │
└────────────────────┬────────────────────────────┬───────────────┘
                     │ HTTP REST                  │ HTTP REST
                     ▼                            ▼
  ┌──────────────────────────────┐  ┌──────────────────────────────┐
  │     ORDER SERVICE            │  │     REPORT SERVICE           │
  │     Puerto 8080              │  │     Puerto 8082              │
  │  ┌────────────────────────┐  │  │  ┌────────────────────────┐  │
  │  │ REST Controllers       │  │  │  │ REST Controllers       │  │
  │  │ Security Interceptors  │  │  │  │ Date Range Filtering   │  │
  │  └────────────────────────┘  │  │  └────────────────────────┘  │
  │           ▼                   │  │           ▲                   │
  │  ┌────────────────────────┐  │  │  ┌────────────────────────┐  │
  │  │ OrderService           │  │  │  │ ReportService          │  │
  │  │ MenuService            │  │  │  │ Aggregate Logic        │  │
  │  │ Validators             │  │  │  └────────────────────────┘  │
  │  └────────────────────────┘  │  │           ▲                   │
  │           ▼                   │  │           │                   │
  │  ┌────────────────────────┐  │  │  ┌────────────────────────┐  │
  │  │ JPA Repositories       │  │  │  │ JPA Repositories       │  │
  │  └────────────────────────┘  │  │  │ (Read-only CQRS)       │  │
  │           ▼                   │  │  └────────────────────────┘  │
  │  ┌────────────────────────┐  │  │           ▼                   │
  │  │ PostgreSQL             │  │  │  ┌────────────────────────┐  │
  │  │ restaurant_db          │  │  │  │ PostgreSQL             │  │
  │  └────────────────────────┘  │  │  │ report_db              │  │
  │           ▼                   │  │  └────────────────────────┘  │
  │  ┌────────────────────────┐  │  │           ▲                   │
  │  │ RabbitMQ Publisher     │  │  │           │ AMQP Consumer     │
  │  └────────────────────────┘  │  │  ┌────────────────────────┐  │
  └────────────┬─────────────────┘  │  │ Event Listener         │  │
               │                     │  └────────────────────────┘  │
               │  AMQP               └────────────────┬──────────────┘
               ▼                                      │
  ┌─────────────────────────────────────────────────┐│
  │             RabbitMQ (Message Broker)            ││
  │  Exchange: order.exchange                        ││
  │  Queues: order.placed.queue                      ││
  │          order.placed.report.queue               ││
  │  DLQ: order.placed.dlq                           ││
  └────────────────────────┬─────────────────────────┘│
                           │                          │
                           │ AMQP Consumer            │ AMQP Consumer
                           ▼                          ▼ (from main)
          ┌──────────────────────────────┐
          │     KITCHEN WORKER           │
          │     (Sin puerto HTTP)        │
          │  ┌────────────────────────┐  │
          │  │ Event Listener         │  │
          │  │ Event Validator        │  │
          │  └────────────────────────┘  │
          │           ▼                   │
          │  ┌────────────────────────┐  │
          │  │ OrderProcessingService │  │
          │  └────────────────────────┘  │
          │           ▼                   │
          │  ┌────────────────────────┐  │
          │  │ JPA Repository         │  │
          │  └────────────────────────┘  │
          │           ▼                   │
          │  ┌────────────────────────┐  │
          │  │ PostgreSQL             │  │
          │  │ kitchen_db             │  │
          │  └────────────────────────┘  │
          └──────────────────────────────┘
                      
External Dependencies (CDN):
  - Unsplash (images.unsplash.com)
  - GitHub Container Registry (ghcr.io)
```

---

## 6. CASOS ESPECIALES O EXCEPCIONES

### Excepción 1: Compatibilidad de Eventos Versionados

**Contexto:** El sistema migró de un esquema de eventos v1 (snake_case) a v2 (camelCase) sin romper consumidores existentes.

**Evento v1 (legacy):**
```json
{
  "event_version": 1,
  "id": "uuid",
  "table_id": 5,
  "items": [...]
}
```

**Evento v2 (actual):**
```json
{
  "eventVersion": 2,
  "orderId": "uuid",
  "tableId": 5,
  "createdAt": "2024-01-15T10:30:00Z",
  "items": [...]
}
```

**Estrategia de compatibilidad:**
- Métodos helper `resolveOrderId()` y `resolveTableId()` en clase de evento
- Fallback a campos v1 si campos v2 son null

**Código de soporte de versiones:**
```java
// OrderPlacedEvent.java (kitchen-worker)
public UUID resolveOrderId() {
    return orderId != null ? orderId : id;
}

public Integer resolveTableId() {
    return tableId != null ? tableId : table_id;
}
```

**Escenario de uso:**
1. Se despliega nueva versión de `order-service` que publica eventos v2
2. `kitchen-worker` antiguo aún puede procesar eventos usando fallback
3. Se actualiza `kitchen-worker` para leer nativamente v2
4. Soporte v1 puede removerse en siguiente major version

**Limitación:** No hay Schema Registry (Confluent/Apicurio) que valide contratos automáticamente.

---

### Excepción 2: Mock Mode para Desarrollo Frontend

**Contexto:** Frontend puede operar sin backend levantado usando datos sintéticos.

**Activación:**
```bash
# .env.local (ambiente desarrollo)
VITE_USE_MOCK=true
```

**Implementación:**
```typescript
// src/api/menu.ts
export async function getMenu(): Promise<Product[]> {
  if (import.meta.env.VITE_USE_MOCK === 'true') {
    return mockMenu() // Datos hardcodeados
  }
  return httpClient.get('/menu') // API real
}
```

**Datos mock incluidos:**
- 12 productos (3 entradas, 4 principales, 3 postres, 2 bebidas)
- Imágenes de placeholder
- Precios en rango $450-$2200

**Propósito:**
- Desarrollo frontend independiente del backend
- Demos sin infraestructura
- Pruebas de UI sin base de datos

**Limitación:**
- Los mocks no replican errores del backend (siempre exitosos)
- Transiciones de estado no persisten entre recargas
- No prueba integración real

**Recomendación:** Usar para prototipos rápidos, pero siempre validar contra API real antes de merge.

---

### Excepción 3: Dead Letter Queue para Eventos Corruptos

**Contexto:** Eventos AMQP con estructura inválida se enrutan a DLQ sin reintentos para evitar loop infinito.

**Validaciones que causan DLQ inmediato:**
1. JSON malformado (syntax error)
2. Campo `eventVersion` faltante
3. Campos requeridos null (`orderId`, `tableId`)
4. Tipo de dato incorrecto (e.g., `tableId` como string en lugar de integer)

**Implementación:**
```java
// OrderEventListener.java (kitchen-worker)
try {
    validator.validate(event);
} catch (InvalidEventContractException ex) {
    log.error("Invalid event contract, sending to DLQ: {}", ex.getMessage());
    throw new AmqpRejectAndDontRequeueException(ex); // DLQ directo
}
```

**Configuración RabbitMQ:**
```yaml
# .env
RABBITMQ_KITCHEN_DLQ_NAME=order.placed.dlq
RABBITMQ_KITCHEN_DLX_NAME=order.dlx
```

**Estrategia de recuperación:**
- Eventos en DLQ se inspeccionan manualmente con RabbitMQ Management UI
- Si el error fue temporal (e.g., despliegue de versión incompatible), se puede:
  1. Mover mensajes de vuelta a la cola principal con shovel
  2. Reprocesar con consumidor adhoc
- Si el evento está genuinamente corrupto, se descarta con log para auditoría

**Métricas ausentes:** No se encontró monitoreo de tamaño de DLQ (debería alertar si crece > 10 mensajes).

---

### Excepción 4: Polling Agresivo en Cocina

**Contexto:** El tablero de cocina ejecuta polling cada 3 segundos para actualizar estados en "tiempo real".

**Implementación:**
```typescript
// KitchenBoardPage.tsx
const POLL_MS = 3000

useEffect(() => {
  async function poll() {
    const orders = await listOrders({ token })
    setOrders(orders)
    timeoutRef.current = window.setTimeout(poll, POLL_MS)
  }
  poll()
  return () => clearTimeout(timeoutRef.current!)
}, [])
```

**Rationale:**
- RabbitMQ no expone notificaciones push al frontend (no hay WebSocket implementado)
- AMQP solo comunica backend-to-backend
- Polling es la estrategia más simple para "simular" tiempo real

**Impacto de performance:**
- Backend recibe 20 req/min por cada instancia de frontend abierta
- Con 5 dispositivos en cocina = 100 req/min (1.67 req/s)
- Carga manejable para microservicio con caché de PostgreSQL

**Alternativa no implementada:**
- WebSocket con Server-Sent Events (SSE)
- Suscripción a eventos de RabbitMQ vía STOMP over WebSocket (requiere plugin adicional)

**Optimización aplicada:**
- El polling se pausa cuando la pestaña del navegador no está visible (uso de Page Visibility API no detectado en código, pero comportamiento esperado de `setTimeout`)

---

### Excepción 5: Operación DELETE Masiva sin Protección

**Contexto:** Endpoint `DELETE /orders` elimina TODOS los pedidos sin confirmación ni filtros.

**Endpoint peligroso:**
```java
// OrderController.java
@DeleteMapping
public ResponseEntity<DeleteAllOrdersResponse> deleteAllOrders() {
    orderService.deleteAllOrders(); // Soft delete de toda la tabla
    return ResponseEntity.ok(new DeleteAllOrdersResponse(
        "All orders marked as deleted"
    ));
}
```

**Request necesario:**
```http
DELETE http://localhost:8080/orders
X-Kitchen-Token: cocina123
```

**Consecuencias:**
1. Todos los pedidos activos desaparecen de tablero de cocina
2. Clientes pierden visibilidad de sus órdenes (404 en `/orders/{id}`)
3. Los datos persisten en BD con flag `deleted = true` pero son irrecuperables vía API

**Ausencias críticas:**
- No requiere confirmación explícita (e.g., query param `?confirm=true`)
- No retorna conteo de registros afectados
- No documenta códigos de error posibles (OpenAPI incomplete)

**Caso de uso legítimo:**
- Limpieza de órdenes de prueba en ambiente staging
- Reset completo al final del día operativo

**Mitigación recomendada:**
```java
@DeleteMapping
public ResponseEntity<DeleteAllOrdersResponse> deleteAllOrders(
    @RequestParam(required = true) String confirm
) {
    if (!"DELETE_ALL_ORDERS".equals(confirm)) {
        throw new InvalidConfirmationException();
    }
    int count = orderService.deleteAllOrders();
    return ResponseEntity.ok(new DeleteAllOrdersResponse(count));
}
```

**Evidencia de riesgo**: Documentado como hallazgo crítico en `docs/week-3-review/REST_API_AUDIT.md`, categoría "Operaciones destructivas sin protección".

---


