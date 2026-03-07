# QA Moderno + Automatización Serenity
## Sistema de Gestión de Pedidos de Restaurante

**Proyecto de Learning:** Automatización de pruebas con BDD, Cucumber y Serenity BDD para un sistema de gestión de pedidos de restaurante.

---

## 📋 Contexto del Proyecto

Este repositorio documenta la estrategia moderna de **Quality Assurance (QA)** y **Automatización de Pruebas** aplicada al **Sistema de Gestión de Pedidos de Restaurante**. El proyecto implementa:

- ✅ **BDD (Behavior-Driven Development)** con enfoque "Los 3 Amigos"
- ✅ **Gherkin** como lenguaje de especificación de casos de prueba
- ✅ **Cucumber** para automatización de escenarios
- ✅ **Serenity BDD** para reportes avanzados de automatización
- ✅ Antipatrones identificados en escritura de casos de prueba

---

## 🎯 Descripción del Sistema

### Objetivo
Desarrollar una plataforma digital integral para la gestión de pedidos en restaurantes que automatice el flujo completo desde la toma del pedido por parte del cliente hasta su preparación en cocina y generación de reportes operacionales.

### Actores Principales
1. **Cliente del Restaurante:** Realiza pedidos a través de la interfaz digital
2. **Personal de Cocina:** Visualiza y gestiona el estado de los pedidos
3. **Cocinero:** Actualiza el progreso de preparación de platos

---

## 🔄 Flujos de Negocio Principales

### **Flujo 1: Creación de Pedido (Cliente)**
1. Selección de mesa (validación: número >= 1)
2. Navegación del menú digital
3. Adición de productos al carrito
4. Revisión y confirmación del pedido
5. Persistencia en backend con UUID único
6. Publicación de evento en RabbitMQ para cocina

### **Flujo 2: Procesamiento en Cocina**
1. Consumo del evento `order.placed` desde RabbitMQ
2. Validación de contrato del evento
3. Actualización de estado: PENDING → IN_PREPARATION → READY

### **Flujo 3: Consulta de Estado en Tiempo Real**
- Cliente monitorea estado del pedido mediante polling/WebSocket
- Estados disponibles: Pendiente, En Preparación, Listo

---

## 📚 Contenido del Repositorio

### Documentos Incluidos

| Archivo | Descripción |
|---------|-------------|
| **BUSINESS_CONTEXT.md** | Análisis detallado del contexto de negocio, flujos críticos y reglas de negocio del sistema |
| **USER_STORIES_REFINEMENT_TABLES.md** | Comparación de historias de usuario originales vs. refinadas con criterios de aceptación en BDD |
| **TEST_CASES_AI.md** | Casos de prueba generados para el sistema utilizando técnicas BDD y Gherkin |
| **EJEMPLO_CUCUMBER.md** | Documentación y ejemplos prácticos de cómo escribir escenarios con Cucumber y Gherkin |
| **BDD, Los 3 Amigos...pdf** | Presentación sobre BDD, metodología "Los 3 Amigos", Gherkin y antipatrones en Cucumber |

---

## 🏗️ Historias de Usuario Refinadas

### **H1a: Selección de Mesa (Cliente)**
- **Descripción:** Identificación de mesa en el menú digital
- **Criterio de Aceptación:** El cliente selecciona su mesa y accede al menú de productos disponibles

### **H1b: Realización de Pedido (Cliente)**
- **Descripción:** Envío del pedido directamente a cocina
- **Criterios de Aceptación:**
  - Visualizar solo productos activos
  - Persistencia del carrito al refrescar
  - Confirmación exitosa con UUID

### **H2: Gestión de Pedidos en Cocina**
- **Descripción:** Visualización y actualización del estado de preparación
- **Criterios de Aceptación:**
  - Listar pedidos pendientes ordenados por tiempo
  - Actualizar estado (PENDING → IN_PREPARATION → READY)

### **H3: Consulta de Estado en Tiempo Real**
- **Descripción:** Monitoreo del progreso del pedido por el cliente
- **Criterios de Aceptación:**
  - Visualizar estado actual del pedido
  - Actualización automática mediante polling

---

## 🧪 Estrategia de Testing

### Técnicas Aplicadas
- ✅ Análisis de valores límite
- ✅ Partición de equivalencia
- ✅ Pruebas negativas
- ✅ Pruebas de persistencia
- ✅ Validación funcional
- ✅ Integración API

### Herramientas
- **Cucumber:** Automatización con Gherkin
- **Serenity BDD:** Reportes avanzados
- **JUnit:** Framework de testing
- **Restassured:** Testing de APIs REST

---

## 🎓 Conceptos Clave

### **BDD (Behavior-Driven Development)**
Metodología que alinea el testing con el comportamiento esperado del negocio, facilitando la comunicación entre desarrolladores, testers y stakeholders.

### **Los 3 Amigos**
Enfoque colaborativo donde:
- **Product Owner/Analyst:** Define requisitos
- **Developer:** Considera implementación técnica
- **QA/Tester:** Diseña casos de prueba

### **Gherkin**
Lenguaje de especificación usando estructura **Dado-Cuando-Entonces (Given-When-Then)** que permite escribir casos de prueba en lenguaje natural.

### **Antipatrones en Cucumber a Evitar**
1. Escenarios demasiado genéricos o vagos
2. Duplicación de pasos entre escenarios
3. Dependencias entre escenarios
4. Exceso de detalles técnicos en el lenguaje natural
5. Falta de mantenibilidad en Step Definitions

---

## 📖 Cómo Usar Este Repositorio

1. **Entender el contexto:** Lee `BUSINESS_CONTEXT.md`
2. **Revisar historias refinadas:** Consulta `USER_STORIES_REFINEMENT_TABLES.md`
3. **Estudiar casos de prueba:** Analiza `TEST_CASES_AI.md`
4. **Aprender Cucumber:** Revisa `EJEMPLO_CUCUMBER.md`
5. **Profundizar en BDD:** Consulta la presentación PDF

---

## 🎯 Objetivos de Learning

Este repositorio es una referencia para:
- Implementación de BDD en proyectos reales
- Escritura efectiva de casos de prueba con Gherkin
- Aplicación de Serenity BDD para automatización
- Evitar antipatrones comunes en testing
- Refinamiento de historias de usuario con criterios de aceptación claros

---

**Versión:** 1.0  
**Última actualización:** 6 de marzo de 2026

