<%*
try {
    // ------------------------------------------------------------------
    // CONFIGURACIÓN
    // ------------------------------------------------------------------

    const RUTA_LIBROS = "Entretenimiento/Libros"; 
    const PROPIEDAD_FECHA = "leido_fin"; 



    // ------------------------------------------------------------------
    // RECOLECCIÓN DE DATOS
    // ------------------------------------------------------------------

    const files = app.vault.getMarkdownFiles();
    
    let biblioteca = [];
    let log = ""; 

    for (let file of files) {
        if (!file.path.startsWith(RUTA_LIBROS)) continue;
        if (file.name.includes("Estadísticas de libros") || file.name.includes("Plantilla")) continue;

        const fileCache = app.metadataCache.getFileCache(file);
        const fm = fileCache?.frontmatter;

        if (!fm) continue; 

        try {
            const paginasRaw = fm.paginas;
            const notaRaw = fm.nota;
            const paginasLeidasRaw = fm.paginas_leidas;
            
            // --- Extraer año de lectura ---
            const fechaRaw = fm[PROPIEDAD_FECHA];
            let anioLeido = null;
            
            if (fechaRaw) {
                const dateObj = new Date(fechaRaw);
                if (!isNaN(dateObj)) {
                    anioLeido = dateObj.getFullYear();
                }
            }
            // -----------------------------

            const datos = {
                titulo: file.basename,
                path: file.path,
                estado: fm.estado ? fm.estado.toString().toLowerCase() : "sin leer",
                tipo: fm.tipo ? fm.tipo.toString().toLowerCase() : "individual",
                formato: fm.formato ? fm.formato.toString().toLowerCase() : "",
                
                paginas: parseInt(paginasRaw) || 0,
                nota: parseFloat(notaRaw) || null,
                paginas_leidas_abandonado: parseInt(paginasLeidasRaw) || 0,
                
                anio: anioLeido, 
                
                favorito: fm.favorito === true,
                autores: fm.autor || [],
                link: `[[${file.basename}|↗]]`
            };

            if (datos.nota !== null) {
                datos.valoracion = datos.nota >= 5 ? "✅" : "❌";
            } else {
                datos.valoracion = datos.favorito ? "✅" : "";
            }
            
            biblioteca.push(datos);

        } catch (errArchivo) {
            log += `Error leyendo archivo ${file.name}: ${errArchivo.message}\n`;
        }
    }

    if (biblioteca.length === 0) {
        throw new Error(`No se han encontrado libros en: '${RUTA_LIBROS}'`);
    }



    // ------------------------------------------------------------------
    // CÁLCULOS GENERALES Y LÓGICA DINÁMICA DE AÑOS
    // ------------------------------------------------------------------
    
    // --- Detección automática del año de corte ---
    // Se busca cuál es el año más antiguo que existe en los libros leídos.
    
    const aniosRegistrados = biblioteca
        .filter(b => b.estado === "leido" && b.anio !== null) // Solo libros leídos y con fecha
        .map(b => b.anio);

    // Se calcula el año mínimo. Si no hay ningún año registrado, se usa el año actual por defecto.
    const minAnioRegistrado = aniosRegistrados.length > 0 
        ? Math.min(...aniosRegistrados) 
        : new Date().getFullYear();

    // Se define la etiqueta dinámica para los libros sin fecha
    // Ej: Si el libro más viejo es de 2021, los sin fecha serán "Antes de 2021"
    const ETIQUETA_SIN_FECHA = `Antes de ${minAnioRegistrado}`;

    // Creación de datos
    let stats = {
        totalLibrosLeidos: 0, totalPaginasLeidas: 0,
        librosIndividuales: 0, librosSagas: 0,
        paginasIndividuales: 0, paginasSagas: 0,
        librosGustados: 0, librosNoGustados: 0,
        librosGustadosIndividuales: 0, librosGustadosSagas: 0,
        librosNoGustadosIndividuales: 0, librosNoGustadosSagas: 0,
        paginasGustadas: 0, paginasNoGustadas: 0,
        librosAbandonados: 0, paginasAbandonadas: 0
    };

    let listaLeidos = [];
    let listaAbandonados = [];
    let sumaPaginasSoloLeidos = 0; 
    
    // Se inicializan los contadores de años
    // Se usa la etiqueta dinámica calculada arriba como clave inicial
    let conteoAnios = {};
    conteoAnios[ETIQUETA_SIN_FECHA] = 0;
    
    let maxLibrosEnUnAnio = 0; 

    // Contadores para años (PÁGINAS)
    let conteoPaginasAnios = {};
    conteoPaginasAnios[ETIQUETA_SIN_FECHA] = 0;
    
    let maxPaginasEnUnAnio = 0;

    // --- Bucle principal de procesamiento ---
    for (let libro of biblioteca) {
        
        // CASO 1: LIBROS LEÍDOS

        if (libro.estado === "leido") {
            stats.totalLibrosLeidos++;
            stats.totalPaginasLeidas += libro.paginas;
            sumaPaginasSoloLeidos += libro.paginas; 
            listaLeidos.push(libro);

            // --- Lógica de Asignación de Años ---
            // Si el libro tiene año, se usa ese año. Si no (es null), se usa la etiqueta "Antes de..."
            let labelAnio = libro.anio ? libro.anio.toString() : ETIQUETA_SIN_FECHA;
            
            // Conteo de Libros por Año
            if (!conteoAnios[labelAnio]) conteoAnios[labelAnio] = 0;
            conteoAnios[labelAnio]++;
            
            // Se actualiza el récord de libros en un año (para la barra de progreso)
            if (conteoAnios[labelAnio] > maxLibrosEnUnAnio) {
                maxLibrosEnUnAnio = conteoAnios[labelAnio];
            }

            // Conteo de Páginas por Año
            if (!conteoPaginasAnios[labelAnio]) conteoPaginasAnios[labelAnio] = 0;
            conteoPaginasAnios[labelAnio] += libro.paginas;

            // Se actualiza el récord de páginas en un año
            if (conteoPaginasAnios[labelAnio] > maxPaginasEnUnAnio) {
                maxPaginasEnUnAnio = conteoPaginasAnios[labelAnio];
            }

            // Clasificación por Tipo y Valoración
            if (libro.tipo === "saga") {
                stats.librosSagas++;
                stats.paginasSagas += libro.paginas;
                if (libro.valoracion === "✅") {
                    stats.librosGustadosSagas++; stats.librosGustados++; stats.paginasGustadas += libro.paginas;
                } else if (libro.valoracion === "❌") {
                    stats.librosNoGustadosSagas++; stats.librosNoGustados++; stats.paginasNoGustadas += libro.paginas;
                }
            } else { 
                stats.librosIndividuales++;
                stats.paginasIndividuales += libro.paginas;
                if (libro.valoracion === "✅") {
                    stats.librosGustadosIndividuales++; stats.librosGustados++; stats.paginasGustadas += libro.paginas;
                } else if (libro.valoracion === "❌") {
                    stats.librosNoGustadosIndividuales++; stats.librosNoGustados++; stats.paginasNoGustadas += libro.paginas;
                }
            }
        } 

        // CASO 2: LIBROS ABANDONADOS

        else if (libro.estado === "abandonado") {
            stats.librosAbandonados++;
            stats.paginasAbandonadas += libro.paginas_leidas_abandonado;
            stats.totalPaginasLeidas += libro.paginas_leidas_abandonado;
            listaAbandonados.push(libro);
        }
    }

    const calcPct = (parcial, total) => total > 0 ? ((parcial / total) * 100).toFixed(1) : "0.0";
    const totalLibrosGlobal = stats.totalLibrosLeidos + stats.librosAbandonados;



    // ------------------------------------------------------------------
    // RANKINGS Y MEDIAS
    // ------------------------------------------------------------------

    const topPaginas = [...listaLeidos, ...listaAbandonados].sort((a, b) => b.paginas - a.paginas);
    
    const mediaPaginas = stats.totalLibrosLeidos > 0 
        ? Math.round(sumaPaginasSoloLeidos / stats.totalLibrosLeidos) 
        : 0;

    const todosConNota = [...listaLeidos, ...listaAbandonados].filter(l => l.nota !== null || l.favorito);
    const topNota = todosConNota.sort((a, b) => {
        if (a.estado !== "abandonado" && b.estado === "abandonado") return -1;
        if (a.estado === "abandonado" && b.estado !== "abandonado") return 1;
        if (a.favorito && !b.favorito) return -1;
        if (!a.favorito && b.favorito) return 1;
        if (b.nota !== a.nota) return b.nota - a.nota;
        return b.paginas - a.paginas;
    });

    let sumaNotas = 0;
    let cuentaNotas = 0;
    [...listaLeidos, ...listaAbandonados].forEach(l => {
        if (l.nota !== null) {
            sumaNotas += l.nota;
            cuentaNotas++;
        }
    });
    const mediaNota = cuentaNotas > 0 ? (sumaNotas / cuentaNotas).toFixed(2) : "0.00";

    const rankingConNota = topNota;
    const rankingSinNota = [...listaLeidos, ...listaAbandonados]
        .filter(l => l.nota === null && !l.favorito)
        .sort((a, b) => {
             if (a.estado !== "abandonado" && b.estado === "abandonado") return -1;
             if (a.estado === "abandonado" && b.estado !== "abandonado") return 1;
             return (b.valoracion === "✅") - (a.valoracion === "✅");
        });



    // ------------------------------------------------------------------
    // SALIDA
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

    // Tabla de libros por año
    r += "###### 📅 Ritmo de lectura\n";
    r += "| Año | Progreso Visual | Cantidad |\n";
    r += "| :--- | :--- | :--- |\n";

    // Se ordenan los años de Mayor a Menor (Descendente)
    // Se fuerza a que la etiqueta "Antes de..." vaya siempre al final
    const aniosOrdenados = Object.keys(conteoAnios).sort((a, b) => {
        if (a === ETIQUETA_SIN_FECHA) return 1;  // Mover al final
        if (b === ETIQUETA_SIN_FECHA) return -1; // Mover al final
        return b - a; // Orden numérico descendente (2025, 2024...)
    });

    for (let anio of aniosOrdenados) {
        const cantidad = conteoAnios[anio];
        const bloques = maxLibrosEnUnAnio > 0 
            ? Math.round((cantidad / maxLibrosEnUnAnio) * 10) 
            : 0;
        
        const barraLlena = "▓".repeat(bloques);
        const barraVacia = "░".repeat(10 - bloques);
        
        r += `| **${anio}** | ${barraLlena}${barraVacia} | **${cantidad}** |\n`;
    }
    r += "\n";

	// --- Páginas ---
    r += "## 📖 Páginas\n\n";
    
    r += `**📄 Páginas totales leídas: ${stats.totalPaginasLeidas}**\n`;
    
    r += `  ↳ 📕 Páginas leídas de libros individuales: ${stats.paginasIndividuales} páginas (${calcPct(stats.paginasIndividuales, stats.totalPaginasLeidas)}%)\n`;
    r += `  ↳ 📚 Páginas leídas de libros de sagas: ${stats.paginasSagas} páginas (${calcPct(stats.paginasSagas, stats.totalPaginasLeidas)}%)\n`;
    r += `  ↳ ✅ Páginas leídas de libros gustados: ${stats.paginasGustadas} (${calcPct(stats.paginasGustadas, stats.totalPaginasLeidas)}%)\n`;
    r += `  ↳ ❌ Páginas de libros no gustados: ${stats.paginasNoGustadas} (${calcPct(stats.paginasNoGustadas, stats.totalPaginasLeidas)}%)\n`;
    r += `  ↳ ❌ Páginas de libros abandonados: ${stats.paginasAbandonadas} (${calcPct(stats.paginasAbandonadas, stats.totalPaginasLeidas)}%)\n`;

    // Tabla de páginas leídas en cada año
    r += "###### 📅 Páginas leídas por año\n";
    r += "| Año | Progreso Visual | Cantidad |\n";
    r += "| :--- | :--- | :--- |\n";

    // Se aplica la misma lógica de ordenación dinámica para las páginas
    const aniosPaginasOrdenados = Object.keys(conteoPaginasAnios).sort((a, b) => {
        if (a === ETIQUETA_SIN_FECHA) return 1; 
        if (b === ETIQUETA_SIN_FECHA) return -1;
        return b - a; 
    });

    for (let anio of aniosPaginasOrdenados) {
        const cantidad = conteoPaginasAnios[anio];
        const bloques = maxPaginasEnUnAnio > 0 
            ? Math.round((cantidad / maxPaginasEnUnAnio) * 10) 
            : 0;
        
        const barraLlena = "▓".repeat(bloques);
        const barraVacia = "░".repeat(10 - bloques);
        
        r += `| **${anio}** | ${barraLlena}${barraVacia} | **${cantidad}** |\n`;
    }

	// Top páginas
    r += "###### 📚 Top de libros por páginas\n";
    
    const getEstrella = (libro) => (libro.favorito && !libro.titulo.includes("⭐")) ? " ⭐" : "";

    topPaginas.forEach((book, index) => {
        let detalle = book.estado === "abandonado" ? ` (${book.paginas_leidas_abandonado} leídas de ${book.paginas} totales)` : `: ${book.paginas} páginas`;
        const simbolo = book.valoracion ? ` ${book.valoracion}` : "";
        r += `${index + 1}. ${book.titulo}${getEstrella(book)}${detalle}${simbolo} ${book.link}\n`;
    });

    r += `\n**📏 Media de longitud de libros leídos: ${mediaPaginas} páginas**\n`;

    // --- APARTADO DE NOTAS ---
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

    tR += r;

} catch (e) {
    tR += "❌ ERROR FATAL DETECTADO ❌\n";
    tR += "Mensaje: " + e.message + "\n";
    tR += "Stack: " + e.stack;
}
%>