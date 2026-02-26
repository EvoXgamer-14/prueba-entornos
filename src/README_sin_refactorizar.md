# LegacyProductProcessor — Código Original (Sin Refactorizar)

> ⚠️ **ADVERTENCIA:** Este código está saturando la memoria del servidor en producción.  
> Se documenta como referencia del estado inicial **antes** de la refactorización.

---

## Descripción

Procesador de productos legado que carga una lista de productos, elimina duplicados y aplica descuentos por categoría. Desarrollado sin considerar escalabilidad ni buenas prácticas de rendimiento.

---

## Problemas Identificados

| # | Problema | Impacto |
|---|----------|---------|
| 1 | **Bucle O(n²)** — doble `for` para detectar duplicados | 🔴 Crítico en Big Data |
| 2 | **Precio negativo no bloqueado a tiempo** — se detecta tarde en el flujo | 🟡 Calidad del dato |
| 3 | **ID duplicado** — sin estructura de búsqueda eficiente | 🔴 Datos inconsistentes |
| 4 | **Mutación directa del objeto** — `pActual.precio = ...` modifica el original | 🟡 Efectos secundarios |
| 5 | **Campos públicos** — sin encapsulación (`String id`, `double precio`...) | 🟡 Mala práctica OOP |
| 6 | **Sin Streams ni programación funcional** — código imperativo difícil de mantener | 🔵 Mantenibilidad |

---

## Estructura del Proyecto

```
LegacyProductProcessor/
├── Producto.java           # Modelo de datos (campos públicos)
└── LegacyProductProcessor.java  # Lógica de procesamiento con O(n²)
```

---

## Código Fuente

```java
import java.util.ArrayList;
import java.util.List;

/**
 * ESTE CÓDIGO ESTÁ SATURANDO LA MEMORIA DEL SERVIDOR EN PRODUCCIÓN.
 * Misión: Usar la IA para refactorizarlo, optimizar el procesamiento de datos masivos
 * y asegurar la rentabilidad del sistema.
 */

class Producto {
    String id;
    String nombre;
    double precio;
    String categoria;

    public Producto(String id, String nombre, double precio, String categoria) {
        this.id = id;
        this.nombre = nombre;
        this.precio = precio;
        this.categoria = categoria;
    }
}

public class LegacyProductProcessor {

    public static void main(String[] args) {
        List<Producto> productosMasivos = new ArrayList<>();

        // Simulación de carga de Big Data (Imagina 10.000 registros aquí)
        productosMasivos.add(new Producto("A1", "Portátil Gaming", 1200.0, "Electrónica"));
        productosMasivos.add(new Producto("A2", "Ratón Pro", -50.0, "Electrónica")); // ERROR: Precio negativo
        productosMasivos.add(new Producto("A1", "Portátil Gaming", 1200.0, "Electrónica")); // ERROR: ID Duplicado
        productosMasivos.add(new Producto("B5", "Silla Oficina", 150.0, "Muebles"));

        long startTime = System.currentTimeMillis();
        List<Producto> resultados = procesarDatos(productosMasivos);
        long endTime = System.currentTimeMillis();

        System.out.println("Tiempo de procesamiento: " + (endTime - startTime) + "ms");
        System.out.println("Total productos limpios: " + resultados.size());
    }

    public static List<Producto> procesarDatos(List<Producto> datos) {
        List<Producto> listaLimpia = new ArrayList<>();

        // BUCLE INEFICIENTE: Complejidad O(n^2)
        for (int i = 0; i < datos.size(); i++) {
            Producto pActual = datos.get(i);
            boolean esDuplicado = false;

            for (int j = 0; j < listaLimpia.size(); j++) {
                if (pActual.id.equals(listaLimpia.get(j).id)) {
                    esDuplicado = true;
                    break;
                }
            }

            if (!esDuplicado) {
                if (pActual.precio > 0) {
                    // Lógica de negocio lenta: modificaciones directas sin Streams
                    if (pActual.categoria.equals("Electrónica")) {
                        pActual.precio = pActual.precio * 0.9; // Aplicar descuento
                    }
                    listaLimpia.add(pActual);
                }
            }
        }
        return listaLimpia;
    }
}
```

---

## Complejidad Algorítmica

```
procesarDatos():  O(n²)  ← por cada elemento, recorre toda listaLimpia
Memoria:          O(n)   ← duplicación de lista completa en RAM
```

---

## Requisitos

- Java 8+
- Sin dependencias externas

---

## Cómo Ejecutar

```bash
javac LegacyProductProcessor.java
java LegacyProductProcessor
```

---

> 📌 Ver [`README_refactorizado.md`](./README_refactorizado.md) para la versión optimizada.
