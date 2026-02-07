```dataviewjs
// 1. Configuración inicial
let carpetaActual = dv.current().file.folder;
let nombreArchivo = "registros"; 
let pagina = dv.pages(`"${carpetaActual}"`).find(p => p.file.name == nombreArchivo);

// Obtener el nombre del mes actual en español y capitalizarlo
let nombreMes = new Date().toLocaleString('es-PE', { month: 'long' });
nombreMes = nombreMes.charAt(0).toUpperCase() + nombreMes.slice(1);

if (!pagina) {
    dv.paragraph("❌ No se encontró el archivo `" + nombreArchivo + ".md` en la carpeta: " + carpetaActual);
} else {
    // 2. Extraer todos los movimientos
    let todosLosMovimientos = pagina.file.lists.where(l => l.Ingreso !== undefined || l.Gasto !== undefined);

    // --- FUNCIONES DE CÁLCULO ---

    function obtenerSaldoInicial(lista, cuenta) {
        return lista
            .filter(m => m.Cuenta == cuenta && m.Categoria == "Inicial")
            .array()
            .reduce((acc, m) => acc + (Number(Array.isArray(m.Ingreso) ? m.Ingreso[0] : m.Ingreso) || 0), 0);
    }

    function obtenerBalanceActual(lista, cuenta) {
        let ingresos = lista.filter(m => m.Cuenta == cuenta && m.Ingreso !== undefined).array()
            .reduce((acc, m) => acc + (Number(Array.isArray(m.Ingreso) ? m.Ingreso[0] : m.Ingreso) || 0), 0);
        
        let gastos = lista.filter(m => m.Cuenta == cuenta && m.Gasto !== undefined).array()
            .reduce((acc, m) => acc + (Number(Array.isArray(m.Gasto) ? m.Gasto[0] : m.Gasto) || 0), 0);
        
        return ingresos - gastos;
    }

    // --- CÁLCULOS POR BANCO ---
    const bancos = [
        { nombre: "BCP", emoji: "🏦" },
        { nombre: "Interbank", emoji: "🏢" },
        { nombre: "Yape", emoji: "📱" }
    ];

    let datosResumen = bancos.map(b => {
        let inicial = obtenerSaldoInicial(todosLosMovimientos, b.nombre);
        let actual = obtenerBalanceActual(todosLosMovimientos, b.nombre);
        return {
            nombre: b.emoji + " " + b.nombre,
            inicial: inicial,
            actual: actual
        };
    });

    // --- 🏛️ SECCIÓN 1: BALANCE GENERAL (Sin Fila Total) ---
    dv.header(1, "🏛️ Balance General");
    
    dv.table(["Cuenta", `Saldo Inicial (${nombreMes})`, "Saldo Disponible"], 
        datosResumen.map(d => [
            d.nombre, 
            "S/ " + d.inicial.toLocaleString('es-PE', {minimumFractionDigits: 2}),
            "**S/ " + d.actual.toLocaleString('es-PE', {minimumFractionDigits: 2}) + "**"
        ])
    );
    
    dv.el("hr", ""); 

    // --- 📝 SECCIÓN 2: DETALLE DE MOVIMIENTOS POR BANCO ---
    function renderizarDetalleBanco(nombreCuenta, emoji) {
        dv.header(2, emoji + " Movimientos " + nombreCuenta);

        let movimientosTabla = todosLosMovimientos
            .filter(m => m.Cuenta == nombreCuenta && m.Categoria !== "Inicial") 
            .array()
            .sort((a, b) => b.Fecha - a.Fecha);

        let filas = movimientosTabla.map(m => {
            let esIng = m.Ingreso !== undefined;
            let v = esIng ? m.Ingreso : m.Gasto;
            let valor = Array.isArray(v) ? v[0] : v;
            return [
                m.Fecha ? String(m.Fecha).slice(0, 10) : "S/F",
                (esIng ? "🟢" : "🔴") + " S/ " + Number(valor).toFixed(2),
                m.Categoria || "Sin Cat.",
                m.Concepto
            ];
        });

        if (filas.length > 0) {
            dv.table(["Fecha", "Monto", "Categoría", "Concepto"], filas);
        } else {
            dv.paragraph("*No hay movimientos nuevos este mes.*");
        }
    }

    bancos.forEach(b => renderizarDetalleBanco(b.nombre, b.emoji));
}
```