# Contexto del Proyecto: Obsidian Reading Stats

Este archivo sirve como **fuente de verdad** y contexto para inteligencias artificiales, agentes y desarrolladores humanos que interactúen con este proyecto. Describe la estructura, la lógica y los requisitos de datos para generar estadísticas de lectura en Obsidian.

---

## 🎯 1. Finalidad del Proyecto
El objetivo principal es **generar un informe estadístico detallado y visual** sobre los hábitos de lectura del usuario basándose en los metadatos de su bóveda de Obsidian.

El sistema escanea todas las notas de la bóveda, identifica aquellas que representan "libros" mediante propiedades específicas, y calcula métricas como:
- Cantidad de libros y páginas leídas.
- Distinción entre libros individuales y sagas.
- Valoración (gustados/no gustados) y notas numéricas.
- Libros abandonados.
- Ritmo de lectura por años (con gráficos de barras ASCII).
- Rankings (Top libros por longitud y por nota).

## 📂 2. Estructura de Archivos

| Archivo | Descripción |
| :--- | :--- |
| **`src/stats.js`** | Contiene la lógica pura de JavaScript. Es el "núcleo" del desarrollo. Se usa para editar y probar la lógica sin la sintaxis de plantillas. |
| **`stats-obsidian.md`** | Es la versión operativa para **Templater**. Contiene el mismo código que `src/stats.js` pero envuelto en etiquetas de ejecución `<%* ... %>`. Al insertar esta plantilla en Obsidian, el código se ejecuta y se sustituye por el reporte Markdown generado. |
| **`docs/DEVELOPMENT.md`** | Este archivo de documentación viva. |
| **`assets/`** | Contiene recursos como plantillas (`templates/`) e imágenes (`img/`). |

> **Nota:** Al modificar la lógica, asegúrate de mantener sincronizados `stats.js` y `stats-obsidian.md`.

## 🧬 3. Modelo de Datos (Frontmatter)
Para que una nota sea reconocida como "Libro" y procesada por el script, debe tener ciertas propiedades en su YAML Frontmatter.

### Propiedades Obligatorias
El script ignora cualquier archivo que no tenga **ambas**:
- `paginas`: (Número) Cantidad total de páginas.
- `estado`: (Texto) Estado de lectura. Valores típicos: `leido`, `abandonado`, `sin leer`.

### Propiedades Completas
| Propiedad | Tipo | Descripción |
| :--- | :--- | :--- |
| `paginas` | `number` | **Requerido**. Longitud del libro. |
| `estado` | `string` | **Requerido**. `leido` (cuenta para stats), `abandonado` (cuenta parcial), otros (ignorado). |
| `leido_fin` | `date` | Fecha de finalización. Usada para agrupar por años (la variable `PROPIEDAD_FECHA` en el código controla esto). |
| `nota` | `number` | Calificación del 0 al 10. |
| `tipo` | `string` | `saga` o `individual`. Afecta a los contadores de categorías. |
| `formato` | `string` | Formato (ej. físico, kindle, audio). (Actualmente se extrae pero no se usa mucho en stats finales). |
| `paginas_leidas`| `number` | Usado para libros **abandonados** (cuántas páginas se leyeron realmente). |
| `favorito` | `boolean`| Si es `true`, marca el libro con ⭐ y afecta al "Gusto" si no hay nota numérica. |
| `autor` | `list` | Lista de autores. |

## ⚙️ 4. Lógica de Funcionamiento Interno

El script sigue un flujo lineal optimizado:

1.  **Inicialización**: Define acumuladores (`stats`, `conteoAnios`, etc.) y configuraciones.
2.  **Escaneo (`app.vault.getMarkdownFiles()`)**:
    *   Itera sobre todos los archivos Markdown.
    *   **Validación Rápida**: Usa `app.metadataCache.getFileCache()` para revisar el Frontmatter sin leer el contenido del archivo (mejora masiva de rendimiento).
    *   **Filtrado**: Descarta archivos sin `paginas`+`estado`, y archivos de sistema (ej. plantillas).
3.  **Extracción y Normalización**:
    *   Crea un objeto `libro` normalizando datos (fechas a años, estados a minúsculas).
    *   Calcula una "Valoración Visual" (✅/❌) basada en si la nota >= 5 o si es favorito.
4.  **Acumulación**:
    *   Si `estado === "leido"`: Suma a totales, contadores por tipo/gusto, y agrupaciones por año.
    *   Si `estado === "abandonado"`: Suma a métricas de abandonados y páginas parciales.
5.  **Cálculo de Finales**:
    *   Calcula medias, porcentajes y ordena las listas (Top Páginas, Top Notas).
    *   Lógica de ordenación compleja: Prioriza leídos > favoritos > nota > páginas.
6.  **Generación de Salida**:
    *   Construye un string Markdown (`r`).
    *   Genera gráficas de barras ASCII (`▓▓▓░░`) para el ritmo anual.
    *   Asigna el resultado final a `tR` (variable de salida de Templater).

## 📊 5. Salida Generada
El reporte resultante contiene:
1.  **Estadísticas Generales**: Totales desglosados (Sagas vs Individuales, Gustados vs No gustados, Abandonados).
2.  **Ritmo de Lectura**: Tabla con barras de progreso por año.
3.  **Páginas**: Totales de páginas y desglose similar a libros, más tabla de páginas por año.
4.  **Top Libros**: Ranking de libros más largos.
5.  **Ranking de Notas**: Todos los libros ordenados por valoración, separando los que tienen nota numérica de los que solo tienen valoración cualitativa.

---
*Este contexto debe actualizarse si se cambian los nombres de las propiedades del Frontmatter, la lógica de cálculo o la estructura de salida.*
