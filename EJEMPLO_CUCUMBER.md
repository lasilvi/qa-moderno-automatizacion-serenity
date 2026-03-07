# Tutorial de Cucumber en 10 minutos (Traducción al español)

## Qué aprenderás

En este tutorial rápido aprenderás a:

- Instalar **Cucumber**
- Escribir tu primer escenario usando **Gherkin**
- Escribir tu primera **Step Definition en Java**
- Ejecutar Cucumber
- Comprender el flujo básico de **BDD (Behaviour‑Driven Development)**

Usaremos Cucumber para desarrollar una pequeña librería que determine **si ya es viernes**.

---

# Requisitos previos

Antes de comenzar necesitas:

- Conocimiento básico de Java
- Experiencia usando la terminal
- Experiencia usando un editor de texto

## Software necesario

- Java SE
- Herramienta de build:
  - Maven 3.3.1 o superior
- IntelliJ IDEA
- Plugin **Cucumber for Java**

Alternativa:

- Eclipse + plugin **Cucumber Eclipse**

---

# Crear un proyecto Cucumber

Ejecuta el siguiente comando en la terminal:

```bash
mvn archetype:generate "-DarchetypeGroupId=io.cucumber" "-DarchetypeArtifactId=cucumber-archetype" "-DarchetypeVersion=7.34.2" "-DgroupId=hellocucumber" "-DartifactId=hellocucumber" "-Dpackage=hellocucumber" "-Dversion=1.0.0-SNAPSHOT" "-DinteractiveMode=false"
```

Resultado esperado:

```text
[INFO] Project created from Archetype in dir: ...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```

Entrar al proyecto:

```bash
cd hellocucumber
```

Abrir en IntelliJ:

```
File → Open → seleccionar pom.xml
Open as Project
```

---

# Verificar instalación

Ejecuta:

```bash
mvn test
```

Salida esperada:

```text
-------------------------------------------------------
T E S T S
-------------------------------------------------------

Tests run: 0, Failures: 0, Errors: 0, Skipped: 0

Results :

Tests run: 0, Failures: 0, Errors: 0, Skipped: 0

[INFO] BUILD SUCCESS
```

Cucumber indica que **no encontró nada para ejecutar**.

---

# Escribir un escenario

En BDD usamos **ejemplos concretos** para especificar el comportamiento del software.

Los escenarios se escriben en archivos `.feature` ubicados en:

```
src/test/resources/hellocucumber
```

Ejemplo de archivo:

```gherkin
Feature: Is it Friday yet?
  Everybody wants to know when it's Friday

  Scenario: Sunday isn't Friday
    Given today is Sunday
    When I ask whether it's Friday yet
    Then I should be told "Nope"
```

Explicación:

- **Feature:** describe la funcionalidad.
- **Scenario:** un ejemplo concreto.
- **Given / When / Then:** pasos del escenario.

---

# Ejecutar el escenario

Ejecuta nuevamente:

```bash
mvn test
```

Cucumber mostrará que el escenario tiene **pasos undefined** y sugerirá código.

Ejemplo de snippets generados:

```java
@Given("today is Sunday")
public void today_is_sunday() {
    throw new io.cucumber.java.PendingException();
}

@When("I ask whether it's Friday yet")
public void i_ask_whether_it_s_friday_yet() {
    throw new io.cucumber.java.PendingException();
}

@Then("I should be told {string}")
public void i_should_be_told(String string) {
    throw new io.cucumber.java.PendingException();
}
```

Copiar estos métodos en:

```
src/test/java/hellocucumber/StepDefinitions.java
```

---

# Escenario pendiente

Si ejecutas nuevamente Cucumber, el escenario aparecerá como **pending**.

Esto significa que:

- Cucumber encontró los métodos
- pero **aún no tienen lógica implementada**

---

# Implementar Step Definitions

Ahora agregamos la lógica:

```java
package hellocucumber;

import io.cucumber.java.en.Given;
import io.cucumber.java.en.When;
import io.cucumber.java.en.Then;
import static org.assertj.core.api.Assertions.assertThat;

class IsItFriday {
    static String isItFriday(String today) {
        return null;
    }
}

public class StepDefinitions {

    private String today;
    private String actualAnswer;

    @Given("today is Sunday")
    public void today_is_Sunday() {
        today = "Sunday";
    }

    @When("I ask whether it's Friday yet")
    public void i_ask_whether_it_s_Friday_yet() {
        actualAnswer = IsItFriday.isItFriday(today);
    }

    @Then("I should be told {string}")
    public void i_should_be_told(String expectedAnswer) {
        assertThat(actualAnswer).isEqualTo(expectedAnswer);
    }
}
```

Si ejecutas las pruebas ahora, el escenario **fallará** porque el método devuelve `null`.

---

# Hacer que el escenario pase

Implementar la lógica mínima:

```java
static String isItFriday(String today) {
    return "Nope";
}
```

Ejecutar nuevamente.

El escenario pasará.

---

# Agregar otro escenario

Actualizar el archivo `.feature`:

```gherkin
Scenario: Friday is Friday
  Given today is Friday
  When I ask whether it's Friday yet
  Then I should be told "TGIF"
```

Agregar Step Definition:

```java
@Given("today is Friday")
public void today_is_Friday() {
    today = "Friday";
}
```

Al ejecutar nuevamente **fallará**, porque aún no implementamos la lógica.

---

# Implementar la lógica real

```java
static String isItFriday(String today) {
    return "Friday".equals(today) ? "TGIF" : "Nope";
}
```

Ahora ambos escenarios pasan.

---

# Usar variables y Examples

Podemos usar **Scenario Outline** para ejecutar el mismo escenario con varios datos.

```gherkin
Scenario Outline: Today is or is not Friday
  Given today is "<day>"
  When I ask whether it's Friday yet
  Then I should be told "<answer>"

Examples:
  | day            | answer |
  | Friday         | TGIF   |
  | Sunday         | Nope   |
  | anything else! | Nope   |
```

Actualizar Step Definitions:

```java
@Given("today is {string}")
public void today_is(String today) {
    this.today = today;
}
```

---

# Refactorización

Con el código funcionando:

- mover `isItFriday()` al código de producción
- extraer métodos reutilizables
- mantener simples las Step Definitions

---

# Resumen

En este tutorial aprendiste:

- Cómo instalar **Cucumber**
- Cómo escribir escenarios con **Gherkin**
- Cómo crear **Step Definitions en Java**
- Cómo ejecutar pruebas BDD
- Cómo usar **Scenario Outline y Examples**

Flujo típico de BDD:

1. Escribir escenario
2. Ejecutar (falla)
3. Implementar pasos
4. Hacer pasar el test
5. Refactorizar
