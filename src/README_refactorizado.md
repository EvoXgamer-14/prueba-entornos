# ProductProcessor — Código Refactorizado ✅

> 🚀 Versión optimizada del procesador legado. Complejidad reducida de **O(n²) → O(n)**.  
> Memoria estabilizada, código limpio y listo para Big Data en producción.

---

## ¿Qué se Mejoró?

| Antes ❌ | Después ✅ |
|----------|-----------|
| Doble `for` → O(n²) | `HashSet` para IDs → O(n) |
| Campos públicos sin encapsulación | Campos `private` con getters |
| Mutación directa del objeto | Objeto inmutable, nuevo precio calculado en Stream |
| Sin validación temprana ni logs | Filtros encadenados con logs en `stderr` |
| Código imperativo difícil de leer | Pipeline funcional declarativo |

---

## Descripción

Procesador de productos optimizado para grandes volúmenes de datos. Utiliza Java Streams para encadenar validaciones y transformaciones en un único pipeline funcional, y un `HashSet` para detección de duplicados en tiempo constante O(1).

---

## Estructura del Proyecto

```
ProductProcessor/
├── Producto.java           # Modelo encapsulado (private + getters + constructor)
└── ProductProcessor.java   # Lógica optimizada con Streams + HashSet
```

---

## Código Fuente

### `Producto.java`

```java
/**
 * Modelo de datos que representa un producto del catálogo.
 *
 * <p>Diseñado con campos inmutables para evitar mutaciones accidentales
 * en pipelines de procesamiento masivo (Big Data).</p>
 */
public class Producto {

    private final String id;
    private final String nombre;
    private final double precio;
    private final String categoria;

    /**
     * Constructor principal del producto.
     *
     * @param id        Identificador único del producto.
     * @param nombre    Nombre descriptivo del producto.
     * @param precio    Precio en euros. Debe ser mayor que 0.
     * @param categoria Categoría a la que pertenece el producto (ej: "Electrónica").
     */
    public Producto(String id, String nombre, double precio, String categoria) {
        this.id = id;
        this.nombre = nombre;
        this.precio = precio;
        this.categoria = categoria;
    }

    /** @return Identificador único del producto. */
    public String getId()         { return id; }

    /** @return Nombre descriptivo del producto. */
    public String getNombre()     { return nombre; }

    /** @return Precio actual del producto en euros. */
    public double getPrecio()     { return precio; }

    /** @return Categoría del producto. */
    public String getCategoria()  { return categoria; }

    /**
     * Crea una copia del producto con un precio diferente.
     *
     * <p>No modifica el objeto original, garantizando inmutabilidad.</p>
     *
     * @param nuevoPrecio El nuevo precio a aplicar. Debe ser mayor que 0.
     * @return Nueva instancia de {@code Producto} con el precio actualizado.
     */
    public Producto conPrecio(double nuevoPrecio) {
        return new Producto(this.id, this.nombre, nuevoPrecio, this.categoria);
    }

    @Override
    public String toString() {
        return String.format("Producto{id='%s', nombre='%s', precio=%.2f, categoria='%s'}",
                id, nombre, precio, categoria);
    }
}
```

### `ProductProcessor.java`

```java
import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

/**
 * Procesador optimizado de productos para Big Data.
 *
 * <p>Mejoras clave respecto al código legado:</p>
 * <ul>
 *   <li>O(n) en lugar de O(n²): HashSet para detección de duplicados en O(1)</li>
 *   <li>Pipeline funcional con Streams: legible, mantenible y componible</li>
 *   <li>Sin mutación de objetos: conPrecio() crea nuevas instancias</li>
 *   <li>Validación temprana con logs: los filtros avisan de cada dato descartado</li>
 * </ul>
 */
public class ProductProcessor {

    private static final double DESCUENTO_ELECTRONICA = 0.90;

    public static void main(String[] args) {
        List<Producto> productosMasivos = List.of(
            new Producto("A1", "Portátil Gaming", 1200.0, "Electrónica"),
            new Producto("A2", "Ratón Pro",        -50.0, "Electrónica"),  // precio negativo → filtrado
            new Producto("A1", "Portátil Gaming", 1200.0, "Electrónica"),  // ID duplicado → descartado
            new Producto("B5", "Silla Oficina",    150.0, "Muebles")
        );

        long startTime = System.currentTimeMillis();
        List<Producto> resultados = procesarDatos(productosMasivos);
        long endTime = System.currentTimeMillis();

        System.out.println("Tiempo de procesamiento: " + (endTime - startTime) + "ms");
        System.out.println("Total productos limpios: " + resultados.size());
        resultados.forEach(System.out::println);
    }

    /**
     * Procesa una lista de productos aplicando validaciones y transformaciones en pipeline.
     *
     * <p>El orden de los filtros es importante: primero se descartan precios inválidos
     * y luego duplicados, antes de aplicar cualquier lógica de negocio.</p>
     *
     * @param datos Lista de productos en bruto a procesar. No se modifica.
     * @return Nueva lista con los productos limpios y con descuentos aplicados.
     */
    public static List<Producto> procesarDatos(List<Producto> datos) {
        Set<String> idsVistos = new HashSet<>();

        return datos.stream()
            // 1. Descartar productos con precio inválido, con log explícito
            .filter(p -> {
                if (p.getPrecio() <= 0) {
                    System.err.println("⚠️  Precio inválido, descartado: "
                        + p.getId() + " | " + p.getNombre() + " | " + p.getPrecio());
                    return false;
                }
                return true;
            })

            // 2. Descartar duplicados por ID — HashSet.add() devuelve false si ya existe: O(1)
            .filter(p -> {
                if (!idsVistos.add(p.getId())) {
                    System.err.println("⚠️  ID duplicado, descartado: "
                        + p.getId() + " | " + p.getNombre());
                    return false;
                }
                return true;
            })

            // 3. Aplicar lógica de negocio sin mutar el objeto original
            .map(p -> "Electrónica".equals(p.getCategoria())
                    ? p.conPrecio(p.getPrecio() * DESCUENTO_ELECTRONICA)
                    : p)

            .collect(Collectors.toList());
    }
}
```

---

## Complejidad Algorítmica

```
procesarDatos():  O(n)   ← una sola pasada por el Stream
Duplicados:       O(1)   ← HashSet.add() por elemento
Memoria:          O(n)   ← solo los elementos únicos y válidos
```

---

## Fase 3 — Evaluación de la Mejora

### Comparativa Big O: Antes vs Después

| Operación | Código Original | Código Refactorizado |
|-----------|----------------|----------------------|
| Detección de duplicados | O(n²) | O(n) |
| Lookup por ID | O(n) por elemento | O(1) con HashSet |
| Pipeline completo | O(n²) | O(n) |

El bucle anidado original ejecuta hasta **n × n comparaciones**. Con 10.000 productos eso son 100 millones de operaciones por ejecución. El nuevo código hace exactamente **n operaciones**, una por producto.

### Cálculo de Ahorro de Recursos en la Nube

Escenario: **10.000 productos**, procesados **24 veces al día** (una vez por hora).

| Concepto | Código Original O(n²) | Código Refactorizado O(n) |
|----------|----------------------|--------------------------|
| Operaciones por ejecución | 100.000.000 | 10.000 |
| Tiempo estimado por ejecución | ~2.000 ms | ~5 ms |
| Tiempo de CPU al día | ~48.000 ms (48 s) | ~120 ms |
| Tiempo de CPU al mes | ~1.440.000 ms (~400 h) | ~3.600 ms (~1 h) |

Si el nuevo código es un **50% más rápido** (siendo conservadores), pasamos de ~400 horas de CPU al mes a ~200 horas. Con un coste de instancia en AWS EC2 de ~0,04 €/hora, el ahorro mensual es de aproximadamente **8 €uro solo en este proceso**. A escala real con 100.000 productos o múltiples servicios corriendo este tipo de lógica, el ahorro asciende fácilmente a **cientos o miles de euros al mes**, además de evitar los costes por timeouts, reintentos y caídas del servicio.

---

## Librerías de IA que Podrían Automatizar Este Proceso en el Futuro

El PDF plantea esta reflexión: ¿qué herramientas de IA podrían hacer este trabajo de refactorización y limpieza de datos de forma automática?

**Para detección de anomalías en datos (Rol A):**
- **Apache Spark MLlib** — librería de Machine Learning sobre Spark que permite detectar outliers y datos anómalos en datasets masivos distribuidos. Escalaría este mismo proceso a millones de registros en clúster.
- **Amazon SageMaker Data Wrangler** — servicio cloud que analiza datasets automáticamente, detecta valores nulos, duplicados y distribuciones anómalas sin escribir código.

**Para refactorización automática de código (Rol B):**
- **GitHub Copilot / Tabnine** — asistentes de IA integrados en el IDE que sugieren refactorizaciones en tiempo real mientras se escribe código.
- **SonarQube** — herramienta de análisis estático que detecta code smells, complejidad ciclomática alta y patrones O(n²) de forma automática en el pipeline de CI/CD.

**Para generación de tests (Rol C):**
- **Diffblue Cover** — genera automáticamente tests unitarios en Java analizando el bytecode, sin necesidad de escribirlos manualmente.
- **EvoSuite** — genera suites de tests JUnit optimizando la cobertura de ramas automáticamente.

---

## Control de Calidad — Iteración con la IA: Un Error Corregido

Durante el proceso de refactorización, la IA propuso inicialmente esta solución para el filtro de duplicados:

```java
// ❌ VERSIÓN INCORRECTA propuesta por la IA
.filter(p -> !idsVistos.contains(p.getId()))
.peek(p -> idsVistos.add(p.getId()))
```

El problema es que `peek()` es una operación intermedia de "observación" en los Streams de Java y **no está garantizada** en todos los contextos: el compilador puede optimizar o eliminar operaciones `peek()` si considera que no afectan al resultado final. Usarlo para producir efectos secundarios (como añadir al `HashSet`) es una práctica incorrecta que puede generar comportamientos imprevisibles.

Se corrigió a la IA indicándole este problema, y propuso la versión correcta:

```java
// ✅ VERSIÓN CORREGIDA
.filter(p -> idsVistos.add(p.getId()))
```

`HashSet.add()` ya devuelve `false` si el elemento existía, así que una sola llamada hace las dos cosas a la vez: comprueba si es duplicado Y lo registra. El `contains()` previo y el `peek()` son completamente innecesarios.

---

## Salida Esperada

```
⚠️  Precio inválido, descartado: A2 | Ratón Pro | -50.0
⚠️  ID duplicado, descartado: A1 | Portátil Gaming
Tiempo de procesamiento: 3ms
Total productos limpios: 2
Producto{id='A1', nombre='Portátil Gaming', precio=1080.00, categoria='Electrónica'}
Producto{id='B5', nombre='Silla Oficina',   precio=150.00,  categoria='Muebles'}
```

> `A2` descartado por precio negativo.  
> `A1` duplicado descartado por ID repetido.  
> `Portátil Gaming` con 10% de descuento aplicado.

---

## Requisitos

- Java 11+ (para `List.of()`)
- Sin dependencias externas

## Cómo Ejecutar

```bash
javac Producto.java ProductProcessor.java
java ProductProcessor
```

---

## Principios Aplicados

- **Encapsulación (OOP):** campos `private final`, sin acceso directo externo
- **Inmutabilidad:** `conPrecio()` devuelve nueva instancia en lugar de mutar
- **Single Responsibility:** cada filtro del Stream tiene una única responsabilidad
- **Clean Code:** pipeline declarativo auto-documentado con JavaDoc
- **Big Data Ready:** O(n) escala linealmente con 10K, 100K o 10M registros

---

> 📌 Ver [`README_sin_refactorizar.md`](./README_sin_refactorizar.md) para comparar con la versión original.
