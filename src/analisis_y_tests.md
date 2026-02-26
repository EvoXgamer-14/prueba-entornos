# Análisis de Código: LegacyProductProcessor

---

## 1. Anomalías Detectadas en los Datos

### Anomalía 1 — Precio Negativo

**Producto afectado:** `"Ratón Pro"` con precio `-50.0`

El producto `A2` tiene un precio negativo, lo cual viola una regla de negocio fundamental. El código original lo filtraba con `precio > 0` de forma silenciosa, sin ningún log ni alerta. En un entorno de producción esto es especialmente peligroso porque **nunca sabrías cuántos registros corruptos se están descartando**.

**Solución aplicada:** Filtro explícito con log en `stderr`:

```java
.filter(p -> {
    if (p.precio <= 0) {
        System.err.println("⚠️  Precio inválido, descartado: " + p.id + " | " + p.nombre + " | " + p.precio);
        return false;
    }
    return true;
})
```

---

### Anomalía 2 — ID Duplicado

**Producto afectado:** `"Portátil Gaming"` con ID `"A1"` registrado dos veces.

El código original detectaba duplicados mediante un bucle anidado con complejidad **O(n²)**. Con 10.000 registros, esto supone hasta **100 millones de comparaciones**, lo que causa la saturación de memoria en producción.

**Solución aplicada:** Uso de `HashSet` para lookup en O(1):

```java
Set<String> idsVistos = new HashSet<>();

.filter(p -> {
    if (!idsVistos.add(p.id)) {
        System.err.println("⚠️  ID duplicado, descartado:  " + p.id + " | " + p.nombre);
        return false;
    }
    return true;
})
```

---

### Anomalía 3 — Mutación Directa del Objeto Original

**Línea problemática:**
```java
pActual.precio = pActual.precio * 0.9; // ⚠️ Modifica el objeto de la lista original
```

El código original modificaba directamente el objeto que vivía en la lista de entrada (`productosMasivos`). Si esa lista se reutiliza en otro punto del sistema, los precios ya estarían alterados **sin ningún aviso**, generando resultados incorrectos difíciles de depurar.

**Solución aplicada:** Crear una copia inmutable del objeto:

```java
public Producto conPrecio(double nuevoPrecio) {
    return new Producto(this.id, this.nombre, nuevoPrecio, this.categoria);
}

// En el stream:
.map(p -> p.categoria.equals("Electrónica") ? p.conPrecio(p.precio * 0.9) : p)
```

---

### Resumen de Anomalías

| # | Tipo | Producto | Impacto |
|---|------|----------|---------|
| 1 | Precio negativo | Ratón Pro (A2) | Dato corrupto descartado en silencio |
| 2 | ID duplicado | Portátil Gaming (A1) | O(n²) → saturación de memoria |
| 3 | Mutación de objeto | Portátil Gaming (A1) | Corrupción silenciosa de datos originales |

---

## 2. Tests Unitarios

Tecnologías usadas: **JUnit 5** + **AssertJ**

### Dependencias `pom.xml`

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.25.3</version>
    <scope>test</scope>
</dependency>
```

---

### Código de Tests

```java
import org.junit.jupiter.api.*;
import org.assertj.core.api.Assertions;
import java.util.*;

class LegacyProductProcessorTest {

    // ─────────────────────────────────────────
    // FIXTURES reutilizables
    // ─────────────────────────────────────────
    private Producto portatil;
    private Producto raton;
    private Producto silla;

    @BeforeEach
    void setUp() {
        portatil = new Producto("A1", "Portátil Gaming", 1200.0, "Electrónica");
        raton    = new Producto("A2", "Ratón Pro",       -50.0,  "Electrónica");
        silla    = new Producto("B5", "Silla Oficina",   150.0,  "Muebles");
    }

    // ─────────────────────────────────────────
    // 1. HAPPY PATH
    // ─────────────────────────────────────────

    @Test
    @DisplayName("Productos válidos y únicos deben procesarse correctamente")
    void deberiaRetornarProductosValidosYUnicos() {
        List<Producto> entrada = List.of(portatil, silla);

        List<Producto> resultado = LegacyProductProcessor.procesarDatos(entrada);

        Assertions.assertThat(resultado)
            .hasSize(2)
            .extracting(p -> p.id)
            .containsExactly("A1", "B5");
    }

    @Test
    @DisplayName("Lista vacía debe retornar lista vacía sin errores")
    void deberiaRetornarListaVaciaSiEntradaEsVacia() {
        List<Producto> resultado = LegacyProductProcessor.procesarDatos(Collections.emptyList());

        Assertions.assertThat(resultado).isEmpty();
    }

    // ─────────────────────────────────────────
    // 2. ANOMALÍA — PRECIO INVÁLIDO
    // ─────────────────────────────────────────

    @Test
    @DisplayName("Producto con precio negativo debe ser descartado")
    void deberiaDescartarProductoConPrecioNegativo() {
        List<Producto> entrada = List.of(raton); // precio -50.0

        List<Producto> resultado = LegacyProductProcessor.procesarDatos(entrada);

        Assertions.assertThat(resultado).isEmpty();
    }

    @Test
    @DisplayName("Producto con precio cero debe ser descartado")
    void deberiaDescartarProductoConPrecioCero() {
        Producto gratis = new Producto("C1", "Producto Gratis", 0.0, "Muebles");

        List<Producto> resultado = LegacyProductProcessor.procesarDatos(List.of(gratis));

        Assertions.assertThat(resultado).isEmpty();
    }

    @Test
    @DisplayName("Solo se descartan los productos inválidos, los válidos permanecen")
    void deberiaDescartarSoloLosInvalidosYMantenerLosValidos() {
        List<Producto> entrada = List.of(portatil, raton, silla);

        List<Producto> resultado = LegacyProductProcessor.procesarDatos(entrada);

        Assertions.assertThat(resultado)
            .hasSize(2)
            .extracting(p -> p.id)
            .containsExactlyInAnyOrder("A1", "B5");
    }

    // ─────────────────────────────────────────
    // 3. ANOMALÍA — ID DUPLICADO
    // ─────────────────────────────────────────

    @Test
    @DisplayName("ID duplicado debe conservar solo la primera ocurrencia")
    void deberiaConservarSoloPrimeraOcurrenciaDeDuplicado() {
        Producto duplicado = new Producto("A1", "Portátil Gaming COPIA", 999.0, "Electrónica");
        List<Producto> entrada = List.of(portatil, duplicado);

        List<Producto> resultado = LegacyProductProcessor.procesarDatos(entrada);

        Assertions.assertThat(resultado).hasSize(1);
        Assertions.assertThat(resultado.get(0).nombre)
            .isEqualTo("Portátil Gaming"); // el original, no la copia
    }

    @Test
    @DisplayName("Múltiples duplicados del mismo ID deben dejar solo uno")
    void deberiaManejarMultiplesDuplicadosDelMismoId() {
        Producto copia1 = new Producto("A1", "Copia 1", 900.0, "Electrónica");
        Producto copia2 = new Producto("A1", "Copia 2", 800.0, "Electrónica");
        List<Producto> entrada = List.of(portatil, copia1, copia2);

        List<Producto> resultado = LegacyProductProcessor.procesarDatos(entrada);

        Assertions.assertThat(resultado).hasSize(1);
    }

    // ─────────────────────────────────────────
    // 4. LÓGICA DE NEGOCIO — DESCUENTO ELECTRÓNICA
    // ─────────────────────────────────────────

    @Test
    @DisplayName("Producto de Electrónica debe recibir descuento del 10%")
    void deberiaAplicarDescuentoDelDiezPorCientoAElectronica() {
        List<Producto> resultado = LegacyProductProcessor.procesarDatos(List.of(portatil));

        Assertions.assertThat(resultado.get(0).precio)
            .isEqualTo(1080.0, Assertions.within(0.01)); // 1200 * 0.9
    }

    @Test
    @DisplayName("Producto que no es Electrónica NO debe recibir descuento")
    void noDeberiaAplicarDescuentoAOtrasCategorias() {
        List<Producto> resultado = LegacyProductProcessor.procesarDatos(List.of(silla));

        Assertions.assertThat(resultado.get(0).precio)
            .isEqualTo(150.0, Assertions.within(0.01));
    }

    // ─────────────────────────────────────────
    // 5. INMUTABILIDAD — no mutar la lista original
    // ─────────────────────────────────────────

    @Test
    @DisplayName("El precio original no debe mutarse al aplicar descuento")
    void noDeberiaMutarElPrecioOriginalDelProducto() {
        double precioOriginal = portatil.precio;

        LegacyProductProcessor.procesarDatos(List.of(portatil));

        Assertions.assertThat(portatil.precio)
            .isEqualTo(precioOriginal); // sigue siendo 1200.0
    }

    // ─────────────────────────────────────────
    // 6. CASO COMBINADO (escenario real de producción)
    // ─────────────────────────────────────────

    @Test
    @DisplayName("Escenario completo: duplicados + precios inválidos + descuento")
    void deberiaGestionarEscenarioCompletoDeProduccion() {
        Producto duplicadoPortatil = new Producto("A1", "Portátil COPIA", 999.0, "Electrónica");
        List<Producto> entrada = List.of(portatil, raton, duplicadoPortatil, silla);

        List<Producto> resultado = LegacyProductProcessor.procesarDatos(entrada);

        // Solo 2 productos válidos y únicos
        Assertions.assertThat(resultado).hasSize(2);

        // El portátil tiene descuento aplicado
        Producto resultadoPortatil = resultado.stream()
            .filter(p -> p.id.equals("A1"))
            .findFirst().orElseThrow();
        Assertions.assertThat(resultadoPortatil.precio)
            .isEqualTo(1080.0, Assertions.within(0.01));

        // La silla mantiene su precio original
        Producto resultadoSilla = resultado.stream()
            .filter(p -> p.id.equals("B5"))
            .findFirst().orElseThrow();
        Assertions.assertThat(resultadoSilla.precio)
            .isEqualTo(150.0, Assertions.within(0.01));
    }
}
```

---

### Cobertura de Tests

| Categoría | Nº Tests | Qué valida |
|-----------|----------|------------|
| Happy path | 2 | Flujo normal sin anomalías |
| Precio inválido | 3 | Negativo, cero, mezcla con válidos |
| ID duplicado | 2 | Primera ocurrencia gana, múltiples copias |
| Lógica de negocio | 2 | Descuento aplicado y no aplicado |
| Inmutabilidad | 1 | El objeto original no se modifica |
| Escenario combinado | 1 | Simulación real de producción |
| **Total** | **11** | |
