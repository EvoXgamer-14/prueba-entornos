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
| Sin validación temprana | Filtros encadenados con Streams |
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
 * Modelo de producto encapsulado.
 * Campos privados, acceso mediante getters, sin mutación externa.
 */
public class Producto {

    private final String id;
    private final String nombre;
    private final double precio;
    private final String categoria;

    public Producto(String id, String nombre, double precio, String categoria) {
        this.id = id;
        this.nombre = nombre;
        this.precio = precio;
        this.categoria = categoria;
    }

    public String getId()         { return id; }
    public String getNombre()     { return nombre; }
    public double getPrecio()     { return precio; }
    public String getCategoria()  { return categoria; }

    /**
     * Devuelve una copia del producto con el precio modificado.
     * No muta el objeto original.
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
 * Mejoras clave:
 *  - O(n) en lugar de O(n²): HashSet para detección de duplicados en O(1)
 *  - Pipeline funcional con Streams: legible, mantenible y componible
 *  - Sin mutación de objetos: conPrecio() crea nuevas instancias
 *  - Validación temprana: los filtros se aplican antes del procesamiento
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
     * Procesa la lista de productos aplicando:
     *  1. Filtro de precios válidos (precio > 0)
     *  2. Eliminación de duplicados por ID en O(1) con HashSet
     *  3. Aplicación de descuentos por categoría sin mutar el original
     *
     * @param datos Lista de productos a procesar
     * @return Lista limpia y transformada
     */
    public static List<Producto> procesarDatos(List<Producto> datos) {
        Set<String> idsVistos = new HashSet<>();

        return datos.stream()
            // 1. Descartar productos con precio inválido
            .filter(p -> p.getPrecio() > 0)

            // 2. Descartar duplicados por ID — HashSet: O(1) por inserción
            .filter(p -> idsVistos.add(p.getId()))

            // 3. Aplicar lógica de negocio sin mutar el objeto original
            .map(p -> {
                if ("Electrónica".equals(p.getCategoria())) {
                    return p.conPrecio(p.getPrecio() * DESCUENTO_ELECTRONICA);
                }
                return p;
            })

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

## Salida Esperada

```
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

---

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
- **Clean Code:** pipeline declarativo auto-documentado
- **Big Data Ready:** O(n) escala linealmente con 10K, 100K o 10M registros

---

> 📌 Ver [`README_sin_refactorizar.md`](./README_sin_refactorizar.md) para comparar con la versión original.
