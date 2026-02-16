<%*
try {
    // ------------------------------------------------------------------
    // CONFIGURACIÓN E INICIALIZACIÓN
    // ------------------------------------------------------------------

    // Nombre de la propiedad Frontmatter usada para la fecha de fin de lectura
    const PROPIEDAD_FECHA = "leido_fin";

    // Almacenes de datos
    let listaLeidos = [];
    let listaAbandonados = [];

    // Objeto principal de estadísticas acumuladas
    // Se calculan al vuelo para evitar recorrer arrays múltiples veces
    let stats = {
        totalLibrosLeidos: 0,
        totalPaginasLeidas: 0,
        librosIndividuales: 0,
        librosSagas: 0,
        paginasIndividuales: 0,
        paginasSagas: 0,
        librosGustados: 0,
        librosNoGustados: 0,
        librosGustadosIndividuales: 0,
        librosGustadosSagas: 0,
        librosNoGustadosIndividuales: 0,
        librosNoGustadosSagas: 0,
        paginasGustadas: 0,
        paginasNoGustadas: 0,
        librosAbandonados: 0,
        paginasAbandonadas: 0
    };

    // Acumuladores auxiliares para medias
    let sumaPaginasSoloLeidos = 0;
    let sumaNotas = 0;
    let cuentaNotas = 0;

    // Estructuras para el desglose por Años
    let conteoAnios = {};          // Cantidad de libros por año numérico
    let conteoPaginasAnios = {};   // Cantidad de páginas por año numérico

    // Contadores temporales para libros sin fecha (se asignarán etiqueta al final)
    let librosSinFecha = 0;
    let paginasSinFecha = 0;

    // Seguimiento del año mínimo para la etiqueta dinámica
    let minAnioRegistrado = new Date().getFullYear(); // Por defecto año actual

    let maxLibrosEnUnAnio = 0;
    let maxPaginasEnUnAnio = 0;

    let log = "";
    const anioActual = new Date().getFullYear();

    // ------------------------------------------------------------------
    // PROCESAMIENTO DE ARCHIVOS
    // ------------------------------------------------------------------
    // Se recorre toda la bóveda pero filtrando por metadatos 'cacheado'
    // Esto es mucho más rápido que leer el contenido de cada archivo.

    const files = app.vault.getMarkdownFiles();

    for (let file of files) {

        // --- Validación Eficiente ---
        // Usamos metadataCache para no leer disco si no es necesario.
        // Identificamos "Libro" por la existencia de 'paginas' y 'estado'.
        const fileCache = app.metadataCache.getFileCache(file);
        const fm = fileCache?.frontmatter;

        if (!fm || !fm.paginas || !fm.estado) continue;

        // Filtros de exclusión explícitos
        if (file.name.includes("Estadísticas de libros") || file.name.includes("Template")) continue;

        try {
            // --- Extracción de Datos ---
            const paginasRaw = fm.paginas;
            const notaRaw = fm.nota;
            const paginasLeidasRaw = fm.paginas_leidas;
            // Normalización de cadenas
            const estado = fm.estado ? fm.estado.toString().toLowerCase() : "sin leer";
            const tipo = fm.tipo ? fm.tipo.toString().toLowerCase() : "individual";
            const formato = fm.formato ? fm.formato.toString().toLowerCase() : "";

            // Procesamos la fecha
            const fechaRaw = fm[PROPIEDAD_FECHA];
            let anioLeido = null;
            if (fechaRaw) {
                const dateObj = new Date(fechaRaw);
                if (!isNaN(dateObj)) {
                    anioLeido = dateObj.getFullYear();
                    // Actualizar año mínimo global
                    if (anioLeido < minAnioRegistrado) minAnioRegistrado = anioLeido;
                }
            }

            // Construimos el objeto libro
            const libro = {
                titulo: file.basename,
                path: file.path,
                estado: estado,
                tipo: tipo,
                formato: formato,
                paginas: parseInt(paginasRaw) || 0,
                nota: parseFloat(notaRaw) || null,
                paginas_leidas_abandonado: parseInt(paginasLeidasRaw) || 0,
                anio: anioLeido,
                favorito: fm.favorito === true,
                autores: fm.autor || [],
                link: `[[${file.basename}|↗]]`,
                valoracion: ""
            };

            // Cálculo de Valoración Visual
            if (libro.nota !== null) {
                libro.valoracion = libro.nota >= 5 ? "✅" : "❌";
            } else {
                libro.valoracion = libro.favorito ? "✅" : "";
            }

            // --- Lógica de Acumulación ---

            if (libro.estado === "leido") {
                listaLeidos.push(libro);

                // Estadísticas Globales
                stats.totalLibrosLeidos++;
                stats.totalPaginasLeidas += libro.paginas;
                sumaPaginasSoloLeidos += libro.paginas;

                // Estadísticas de Nota
                if (libro.nota !== null) {
                    sumaNotas += libro.nota;
                    cuentaNotas++;
                }

                // Clasificación por Tipo y Gusto
                const esSaga = (libro.tipo === "saga");
                const leGusto = (libro.valoracion === "✅");

                if (esSaga) {
                    stats.librosSagas++;
                    stats.paginasSagas += libro.paginas;
                    if (leGusto) {
                        stats.librosGustadosSagas++;
                        stats.librosGustados++;
                        stats.paginasGustadas += libro.paginas;
                    } else {
                        stats.librosNoGustadosSagas++;
                        stats.librosNoGustados++;
                        stats.paginasNoGustadas += libro.paginas;
                    }
                } else {
                    stats.librosIndividuales++;
                    stats.paginasIndividuales += libro.paginas;
                    if (leGusto) {
                        stats.librosGustadosIndividuales++;
                        stats.librosGustados++;
                        stats.paginasGustadas += libro.paginas;
                    } else {
                        stats.librosNoGustadosIndividuales++;
                        stats.librosNoGustados++;
                        stats.paginasNoGustadas += libro.paginas;
                    }
                }

                // Acumulación por Años (para gráficas)
                if (anioLeido) {
                    let strAnio = anioLeido.toString();

                    conteoAnios[strAnio] = (conteoAnios[strAnio] || 0) + 1;
                    if (conteoAnios[strAnio] > maxLibrosEnUnAnio) maxLibrosEnUnAnio = conteoAnios[strAnio];

                    conteoPaginasAnios[strAnio] = (conteoPaginasAnios[strAnio] || 0) + libro.paginas;
                    if (conteoPaginasAnios[strAnio] > maxPaginasEnUnAnio) maxPaginasEnUnAnio = conteoPaginasAnios[strAnio];
                } else {
                    librosSinFecha++;
                    paginasSinFecha += libro.paginas;
                }

            } else if (libro.estado === "abandonado") {
                listaAbandonados.push(libro);

                stats.librosAbandonados++;
                stats.paginasAbandonadas += libro.paginas_leidas_abandonado;
                stats.totalPaginasLeidas += libro.paginas_leidas_abandonado;

                // Las notas de libros abandonados también cuentan para la media global
                if (libro.nota !== null) {
                    sumaNotas += libro.nota;
                    cuentaNotas++;
                }
            }

        } catch (err) {
            log += `Error procesando ${file.name}: ${err.message}\n`;
        }
    }

    // ------------------------------------------------------------------
    // FINALIZACIÓN DE DATOS
    // ------------------------------------------------------------------

    if (listaLeidos.length === 0 && listaAbandonados.length === 0) {
        throw new Error("No se han encontrado libros válidos (con 'paginas' y 'estado') en toda la bóveda.");
    }

    // Integrar libros sin fecha con la etiqueta dinámica correcta
    const ETIQUETA_SIN_FECHA = `Antes de ${minAnioRegistrado}`;

    if (librosSinFecha > 0) {
        conteoAnios[ETIQUETA_SIN_FECHA] = librosSinFecha;
        if (librosSinFecha > maxLibrosEnUnAnio) maxLibrosEnUnAnio = librosSinFecha;
    }

    if (paginasSinFecha > 0) {
        conteoPaginasAnios[ETIQUETA_SIN_FECHA] = paginasSinFecha;
        if (paginasSinFecha > maxPaginasEnUnAnio) maxPaginasEnUnAnio = paginasSinFecha;
    }

    // Cálculo de Medias
    const calcPct = (parcial, total) => total > 0 ? ((parcial / total) * 100).toFixed(1) : "0.0";
    const totalLibrosGlobal = stats.totalLibrosLeidos + stats.librosAbandonados;
    const mediaPaginas = stats.totalLibrosLeidos > 0 ? Math.round(sumaPaginasSoloLeidos / stats.totalLibrosLeidos) : 0;
    const mediaNota = cuentaNotas > 0 ? (sumaNotas / cuentaNotas).toFixed(2) : "0.00";

    // Unificación de listas para rankings
    const todosLibros = [...listaLeidos, ...listaAbandonados];

    // Top Páginas (Ordenar descendente)
    const topPaginas = [...todosLibros].sort((a, b) => b.paginas - a.paginas);

    // Top Nota (Lógica compleja de ordenación)
    const rankingConNota = todosLibros
        .filter(l => l.nota !== null || l.favorito)
        .sort((a, b) => {
            if (a.estado !== "abandonado" && b.estado === "abandonado") return -1; // Leídos primero
            if (a.estado === "abandonado" && b.estado !== "abandonado") return 1;
            if (a.favorito && !b.favorito) return -1; // Favoritos primero
            if (!a.favorito && b.favorito) return 1;
            if (b.nota !== a.nota && a.nota !== null && b.nota !== null) return b.nota - a.nota; // Mayor nota
            return b.paginas - a.paginas; // Más páginas desempata
        });

    const rankingSinNota = todosLibros
        .filter(l => l.nota === null && !l.favorito)
        .sort((a, b) => {
            if (a.estado !== "abandonado" && b.estado === "abandonado") return -1;
            if (a.estado === "abandonado" && b.estado !== "abandonado") return 1;
            return (b.valoracion === "✅") - (a.valoracion === "✅"); // Gustados primero
        });

    // ------------------------------------------------------------------
    // GENERACIÓN DE SALIDA (MARKDOWN)
    // ------------------------------------------------------------------

    let r = "";
    if (log !== "") r += `⚠️ ADVERTENCIAS:\n${log}\n\n`;

    r += "## Estadísticas generales\n\n";
    r += `**📚 Libros leídos: ${stats.totalLibrosLeidos}**\n`;
    r += `  ↳ 📕 Libros individuales: ${stats.librosIndividuales} (${calcPct(stats.librosIndividuales, stats.totalLibrosLeidos)}%)\n`;
    r += `  ↳ 📚 Libros de sagas: ${stats.librosSagas} (${calcPct(stats.librosSagas, stats.totalLibrosLeidos)}%)\n`;
    r += `  ↳ ✅ Libros que me han gustado: ${stats.librosGustados} (${calcPct(stats.librosGustados, stats.totalLibrosLeidos)}%)\n`;
    r += `   ↳ Libros individuales que me han gustado: ${stats.librosGustadosIndividuales}\n`;
    r += `   ↳ Libros de sagas que me han gustado: ${stats.librosGustadosSagas}\n`;
    r += `  ↳ ❌ Libros que no me han gustado: ${stats.librosNoGustados} (${calcPct(stats.librosNoGustados, stats.totalLibrosLeidos)}%)\n`;
    r += `   ↳ Libros individuales que no me han gustado: ${stats.librosNoGustadosIndividuales}\n`;
    r += `   ↳ Libros de sagas que no me han gustado: ${stats.librosNoGustadosSagas}\n`;
    r += `  ↳ ❌ Libros abandonados: ${stats.librosAbandonados} (${calcPct(stats.librosAbandonados, totalLibrosGlobal)}% del total de intentos)\n`;

    // Tabla: Ritmo de lectura
    r += "###### 📅 Ritmo de lectura\n";
    r += "| Año | Progreso Visual | Cantidad |\n";
    r += "| :--- | :--- | :--- |\n";

    // Ordenar años descendente, con "Antes de..." al final
    const aniosOrdenados = Object.keys(conteoAnios).sort((a, b) => {
        if (a === ETIQUETA_SIN_FECHA) return 1;
        if (b === ETIQUETA_SIN_FECHA) return -1;
        return b - a;
    });

    for (let anio of aniosOrdenados) {
        const cantidad = conteoAnios[anio];
        const bloques = maxLibrosEnUnAnio > 0 ? Math.round((cantidad / maxLibrosEnUnAnio) * 10) : 0;
        const barrilete = "▓".repeat(bloques) + "░".repeat(10 - bloques);
        r += `| **${anio}** | ${barrilete} | **${cantidad}** |\n`;
    }
    r += "\n";

    // Sección Páginas
    r += "## 📖 Páginas\n\n";
    r += `**📄 Páginas totales leídas: ${stats.totalPaginasLeidas}**\n`;
    r += `  ↳ 📕 Páginas leídas de libros individuales: ${stats.paginasIndividuales} páginas (${calcPct(stats.paginasIndividuales, stats.totalPaginasLeidas)}%)\n`;
    r += `  ↳ 📚 Páginas leídas de libros de sagas: ${stats.paginasSagas} páginas (${calcPct(stats.paginasSagas, stats.totalPaginasLeidas)}%)\n`;
    r += `  ↳ ✅ Páginas leídas de libros gustados: ${stats.paginasGustadas} (${calcPct(stats.paginasGustadas, stats.totalPaginasLeidas)}%)\n`;
    r += `  ↳ ❌ Páginas de libros no gustados: ${stats.paginasNoGustadas} (${calcPct(stats.paginasNoGustadas, stats.totalPaginasLeidas)}%)\n`;
    r += `  ↳ ❌ Páginas de libros abandonados: ${stats.paginasAbandonadas} (${calcPct(stats.paginasAbandonadas, stats.totalPaginasLeidas)}%)\n`;

    // Tabla: Páginas por año
    r += "###### 📅 Páginas leídas por año\n";
    r += "| Año | Progreso Visual | Cantidad |\n";
    r += "| :--- | :--- | :--- |\n";

    const aniosPaginasOrdenados = Object.keys(conteoPaginasAnios).sort((a, b) => {
        if (a === ETIQUETA_SIN_FECHA) return 1;
        if (b === ETIQUETA_SIN_FECHA) return -1;
        return b - a;
    });

    for (let anio of aniosPaginasOrdenados) {
        const cantidad = conteoPaginasAnios[anio];
        const bloques = maxPaginasEnUnAnio > 0 ? Math.round((cantidad / maxPaginasEnUnAnio) * 10) : 0;
        const barrilete = "▓".repeat(bloques) + "░".repeat(10 - bloques);
        r += `| **${anio}** | ${barrilete} | **${cantidad}** |\n`;
    }

    // Top Libros
    r += "###### 📚 Top de libros por páginas\n";

    const getEstrella = (libro) => (libro.favorito && !libro.titulo.includes("⭐")) ? " ⭐" : "";

    topPaginas.forEach((book, index) => {
        let detalle = book.estado === "abandonado" ? ` (${book.paginas_leidas_abandonado} leídas de ${book.paginas} totales)` : `: ${book.paginas} páginas`;
        const simbolo = book.valoracion ? ` ${book.valoracion}` : "";
        r += `${index + 1}. ${book.titulo}${getEstrella(book)}${detalle}${simbolo} ${book.link}\n`;
    });

    r += `\n**📏 Media de longitud de libros leídos: ${mediaPaginas} páginas**\n`;

    // Notas Finales
    r += "\n## 📝 Notas\n\n";
    r += "###### 🏆 Top por nota\n";

    let pos = 0;
    rankingConNota.forEach((book) => {
        pos++;
        const notaTexto = book.nota !== null ? `: ${book.nota}` : "";
        const simbolo = book.valoracion ? ` ${book.valoracion}` : "";
        const extraAbandonado = book.estado === "abandonado" ? " (Abandonado)" : "";
        r += `${pos}. ${book.titulo}${getEstrella(book)}${notaTexto}${simbolo}${extraAbandonado} ${book.link}\n`;
    });

    rankingSinNota.forEach((book) => {
        pos++;
        const simbolo = book.valoracion ? ` ${book.valoracion}` : "";
        const extraAbandonado = book.estado === "abandonado" ? " (Abandonado)" : "";
        r += `${pos}. ${book.titulo}${simbolo}${extraAbandonado} ${book.link}\n`;
    });

    r += `\n**⭐ Nota media de todas las lecturas: ${mediaNota}**\n`;

    // Asignación final a la variable de salida de Templater
    tR += r;

} catch (e) {
    tR += "❌ ERROR FATAL DETECTADO ❌\n";
    tR += "Mensaje: " + e.message + "\n";
    tR += "Stack: " + e.stack;
}
%>