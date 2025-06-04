/****************************
 * CONFIGURACIÓN GLOBAL
 ***************************/
const CONFIG = {
  hojaDatos: "BBDD",
  hojaListaArticulos: "Lista",
  hojaCorreos: "CORREOS_TO_CC",
  columnas: {
    id: 1,                  // A
    fechaCreacion: 2,       // B (NUEVO)
    fechaUltimoEstado: 3,   // C (Antes era 'fecha' en B)
    email: 4,               // D
    area: 5,                // E
    tipo: 6,                // F
    numFactura: 7,          // G
    razonSocial: 8,         // H
    monto: 9,               // I
    articulo: 10,           // J (Ahora contendrá el artículo original o el alternativo)
    estado: 11              // K
    // Total: 11 columnas
  }
};

/****************************
 * FUNCIONES DE ROLES
 ***************************/
function getUserRole(email) { 
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const hojaCorreos = ss.getSheetByName(CONFIG.hojaCorreos); // [cite: 77]
  const encargadas = hojaCorreos.getRange("D2:D").getValues().flat().filter(String); // [cite: 78]
  if (encargadas.includes(email)) return "encargada"; // [cite: 78]
  const practicantes = hojaCorreos.getRange("C2:C").getValues().flat().filter(String); // [cite: 79]
  if (practicantes.includes(email)) return "practicante"; // [cite: 79]
  const asesores = hojaCorreos.getRange("B2:B").getValues().flat().filter(String); // [cite: 80]
  if (asesores.includes(email)) return "asesor"; // [cite: 80]
  throw new Error("Usuario no autorizado"); // [cite: 81]
}

/****************************
 * FUNCIONES PRINCIPALES
 ***************************/
function doGet() {
  return HtmlService.createTemplateFromFile('Index') // [cite: 6]
    .evaluate() // [cite: 6]
    .setTitle('Sistema de Canjes') // [cite: 6]
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL); // [cite: 6]
}

function include(filename) {
  return HtmlService.createHtmlOutputFromFile(filename).getContent(); // [cite: 7]
}

/****************************
 * FUNCIONES DE DATOS
 ***************************/
function getDatosUsuario() {
  const user = getUserData();
  // Logger.log("getDatosUsuario - Usuario: " + JSON.stringify(user));

  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const hoja = ss.getSheetByName(CONFIG.hojaDatos);
  if (!hoja) {
    Logger.log("Error: No se encontró la hoja de datos: " + CONFIG.hojaDatos);
    return [];
  }
  const datos = hoja.getDataRange().getValues();
  // Logger.log("getDatosUsuario - Datos crudos de " + CONFIG.hojaDatos + ": " + datos.length + " filas.");

  const solicitudes = datos.slice(1).filter(fila => {
    const id = fila[CONFIG.columnas.id - 1];
    if (!id || String(id).trim() === "") return false; // Omitir filas sin ID válido

    if (user.role === "encargada" || user.role === "practicante") return true;
    return fila[CONFIG.columnas.email - 1] === user.email;
  }).map(fila => {
    try {
      const parseDate = (dateValue) => {
        if (dateValue instanceof Date) return dateValue.toISOString();
        if (dateValue) {
          const d = new Date(dateValue);
          if (!isNaN(d.getTime())) return d.toISOString();
        }
        return null;
      };

      return {
        id: fila[CONFIG.columnas.id - 1],
        fechaCreacion: parseDate(fila[CONFIG.columnas.fechaCreacion - 1]),
        fechaUltimoEstado: parseDate(fila[CONFIG.columnas.fechaUltimoEstado - 1]),
        email: String(fila[CONFIG.columnas.email - 1] || ""),
        area: String(fila[CONFIG.columnas.area - 1] || ""),
        tipo: String(fila[CONFIG.columnas.tipo - 1] || ""),
        numFactura: String(fila[CONFIG.columnas.numFactura - 1] || ""),
        razonSocial: String(fila[CONFIG.columnas.razonSocial - 1] || ""),
        monto: fila[CONFIG.columnas.monto - 1] !== undefined && fila[CONFIG.columnas.monto - 1] !== null && fila[CONFIG.columnas.monto - 1] !== "" ? parseFloat(fila[CONFIG.columnas.monto - 1]) : null,
        articulo: String(fila[CONFIG.columnas.articulo - 1] || ""),
        estado: String(fila[CONFIG.columnas.estado - 1] || "Desconocido")
      };
    } catch (e) {
      Logger.log("Error al mapear fila en getDatosUsuario: " + e.toString() + " Fila: " + JSON.stringify(fila.slice(0, CONFIG.columnas.estado)));
      return null;
    }
  }).filter(s => s !== null);
  
  // Logger.log("getDatosUsuario - Solicitudes procesadas (" + solicitudes.length + "): " + JSON.stringify(solicitudes.slice(0,2))); // Loguear solo las primeras para no exceder cuota
  return solicitudes;
}

function obtenerSolicitudPorId(solicitudId) { // Modificado para 11 columnas
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const hoja = ss.getSheetByName(CONFIG.hojaDatos);
  if (!hoja) {
    Logger.log("Error en obtenerSolicitudPorId: No se encontró la hoja de datos: " + CONFIG.hojaDatos);
    throw new Error("Hoja de datos no encontrada.");
  }
  const datos = hoja.getDataRange().getValues();
  const c = CONFIG.columnas;

  for (let i = 1; i < datos.length; i++) {
    if (datos[i][c.id - 1] == solicitudId) {
      const parseDate = (dateValue) => { /* ... (misma función parseDate que en getDatosUsuario) ... */ 
          if (dateValue instanceof Date) return dateValue.toISOString();
          if (dateValue) { const d = new Date(dateValue); if (!isNaN(d.getTime())) return d.toISOString(); }
          return null;
      };
      return {
        id: datos[i][c.id - 1],
        fechaCreacion: parseDate(datos[i][c.fechaCreacion - 1]),
        fechaUltimoEstado: parseDate(datos[i][c.fechaUltimoEstado - 1]),
        email: String(datos[i][c.email - 1] || ""),
        area: String(datos[i][c.area - 1] || ""),
        tipo: String(datos[i][c.tipo - 1] || ""),
        numFactura: String(datos[i][c.numFactura - 1] || ""),
        razonSocial: String(datos[i][c.razonSocial - 1] || ""),
        monto: datos[i][c.monto - 1] !== undefined && datos[i][c.monto - 1] !== null && datos[i][c.monto - 1] !== "" ? parseFloat(datos[i][c.monto - 1]) : null,
        articulo: String(datos[i][c.articulo - 1] || ""),
        estado: String(datos[i][c.estado - 1] || "Desconocido")
      };
    }
  }
  throw new Error("Solicitud con ID " + solicitudId + " no encontrada.");
}


/****************************
 * FUNCIONES DE ARTÍCULOS
 ***************************/
function getArticulosDisponibles(area, montoMaximo, articuloExcluido = null) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const hoja = ss.getSheetByName(CONFIG.hojaListaArticulos);
  if (!hoja) {
    Logger.log("Error en getArticulosDisponibles: No se encontró la hoja: " + CONFIG.hojaListaArticulos);
    return [];
  }
  const datos = hoja.getRange("A2:E" + Math.max(2, hoja.getLastRow())).getValues(); 
  
  if (!area || montoMaximo === undefined || montoMaximo === null || montoMaximo < 0) return [];
  const montoNum = parseFloat(montoMaximo);
  if (isNaN(montoNum)) return [];

  const columnaMonto = area === "Repuesto" ? 3 : 4; 
  
  return datos
    .filter(fila => {
      const disponible = fila[0] === true;
      const montoArticulo = parseFloat(fila[columnaMonto]);
      const nombreArticulo = String(fila[1] || "").trim();
      const noExcluido = articuloExcluido ? nombreArticulo !== String(articuloExcluido).trim() : true;
      
      return disponible && !isNaN(montoArticulo) && montoArticulo <= montoNum && noExcluido;
    })
    .map(fila => ({
      nombre: String(fila[1] || ""),
      puntos: fila[2], // Asumiendo que puntos es la columna C
      monto: parseFloat(fila[columnaMonto])
    }))
    .sort((a, b) => b.monto - a.monto);
}

function getArticulosConDisponibilidad() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const hoja = ss.getSheetByName(CONFIG.hojaListaArticulos);
  if (!hoja) {
    Logger.log("Error en getArticulosConDisponibilidad: No se encontró la hoja: " + CONFIG.hojaListaArticulos);
    return [];
  }
  // Asumimos: Col A: Disponibilidad (TRUE/FALSE), Col B: Nombre Artículo, Col D: Monto Repuesto, Col E: Monto Taller
  const datos = hoja.getRange("A2:E" + Math.max(2, hoja.getLastRow())).getValues();
  
  return datos.map(fila => ({
    nombre: String(fila[1] || ""), // Columna B - Nombre
    disponible: fila[0] === true,    // Columna A - Disponibilidad
    montoRepuesto: fila[3] !== undefined && fila[3] !== null && fila[3] !== "" ? parseFloat(fila[3]) : null, // Columna D
    montoTaller: fila[4] !== undefined && fila[4] !== null && fila[4] !== "" ? parseFloat(fila[4]) : null    // Columna E
  })).filter(art => art.nombre !== ""); // No incluir filas sin nombre de artículo
}

function toggleDisponibilidadArticulo(nombreArticulo) {
  if (!nombreArticulo) throw new Error("Nombre de artículo no proporcionado.");
  const user = getUserData();
  if (user.role !== 'encargada') throw new Error("Acción no autorizada.");

  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const hoja = ss.getSheetByName(CONFIG.hojaListaArticulos);
  if (!hoja) throw new Error("Hoja de lista de artículos no encontrada.");

  const textFinder = hoja.createTextFinder(String(nombreArticulo).trim());
  const ranges = textFinder.findAll();

  if (ranges.length === 0) throw new Error("Artículo '" + nombreArticulo + "' no encontrado en la columna B.");
  
  let actualizado = false;
  ranges.forEach(range => {
    // Verificar que el hallazgo esté en la columna B (índice 2)
    if (range.getColumn() === 2) {
      const fila = range.getRow();
      const celdaDisponibilidad = hoja.getRange(fila, 1); // Columna A
      const valorActual = celdaDisponibilidad.getValue();
      celdaDisponibilidad.setValue(valorActual !== true); // Invierte el valor booleano
      actualizado = true;
      Logger.log("Disponibilidad de '" + nombreArticulo + "' cambiada a " + (valorActual !== true) + " por " + user.email);
    }
  });

  if (!actualizado) throw new Error("Artículo '" + nombreArticulo + "' encontrado, pero no en la columna esperada (B).");
  return { success: true, nombre: nombreArticulo };
}


function crearSolicitud(formData) { // Modificado para 11 columnas
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const hoja = ss.getSheetByName(CONFIG.hojaDatos);
  if (!hoja) throw new Error("Hoja de datos no encontrada en crearSolicitud.");

  const lastRow = hoja.getLastRow();
  const lastId = lastRow > 0 ? hoja.getRange(lastRow, CONFIG.columnas.id).getValue() : 0;
  const newId = (typeof lastId === 'number' && lastId > 0) ? lastId + 1 : (hoja.getLastRow() === 0 ? 1 : lastId + 1); // Manejar hoja vacía

  const now = new Date();
  const newRow = [];
  newRow[CONFIG.columnas.id - 1] = newId;
  newRow[CONFIG.columnas.fechaCreacion - 1] = now;
  newRow[CONFIG.columnas.fechaUltimoEstado - 1] = now;
  newRow[CONFIG.columnas.email - 1] = Session.getActiveUser().getEmail();
  newRow[CONFIG.columnas.area - 1] = formData.area;
  newRow[CONFIG.columnas.tipo - 1] = formData.tipo;
  newRow[CONFIG.columnas.numFactura - 1] = formData.numFactura;
  newRow[CONFIG.columnas.razonSocial - 1] = formData.razonSocial;
  newRow[CONFIG.columnas.monto - 1] = parseFloat(formData.monto);
  newRow[CONFIG.columnas.articulo - 1] = formData.articulo;
  newRow[CONFIG.columnas.estado - 1] = "Creado";
  
  // Asegurar que el array tenga 11 elementos, rellenando con "" si es necesario
  // Esto es importante si CONFIG.columnas tiene índices > 11 pero que ya no se usan.
  // Sin embargo, con el nuevo CONFIG, la última columna es la 11.
  const finalRow = [];
    for (let i = 0; i < CONFIG.columnas.estado; i++) { // CONFIG.columnas.estado es el índice más alto (11)
        finalRow[i] = newRow[i] !== undefined ? newRow[i] : "";
    }

  hoja.appendRow(finalRow);
  Logger.log("Nueva solicitud creada ID: " + newId + " por " + Session.getActiveUser().getEmail());
  return newId;
}

/****************************
 * FUNCIONES DE ESTADOS
 ***************************/
function actualizarEstado(id, nuevoEstado, opciones = {}) {
  const { nuevoArticulo = null } = opciones; // opciones.nuevoArticulo
  const user = getUserData(); // Para loguear quién hizo el cambio

  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const hoja = ss.getSheetByName(CONFIG.hojaDatos);
  if (!hoja) throw new Error("Hoja de datos no encontrada en actualizarEstado.");
  
  const datos = hoja.getRange(1, 1, hoja.getLastRow(), CONFIG.columnas.estado).getValues(); // Leer hasta la última columna definida
  let filaActualizada = -1;

  for (let i = 1; i < datos.length; i++) { // Empezar en 1 si la fila 0 es encabezado
    if (datos[i][CONFIG.columnas.id - 1] == id) {
      hoja.getRange(i + 1, CONFIG.columnas.estado).setValue(nuevoEstado);
      hoja.getRange(i + 1, CONFIG.columnas.fechaUltimoEstado).setValue(new Date());
      
      if (nuevoArticulo) {
        hoja.getRange(i + 1, CONFIG.columnas.articulo).setValue(nuevoArticulo);
      }
      filaActualizada = i + 1;
      break; 
    }
  }

  if (filaActualizada === -1) {
    throw new Error("Solicitud no encontrada para actualizar: ID " + id);
  }
  
  Logger.log("Estado de solicitud ID " + id + " actualizado a '" + nuevoEstado + "'" + 
             (nuevoArticulo ? " con nuevo artículo '" + nuevoArticulo + "'" : "") + 
             " por " + user.email);

  // Las notificaciones por email están desactivadas por ahora.
  // Si se implementan notificaciones en la app, aquí se podría marcar la solicitud.
  // enviarNotificacion(id, nuevoEstado); 
  
  return { success: true, message: "Estado actualizado correctamente." };
}


/****************************
 * NOTIFICACIONES
 ***************************/
function generarCuerpoEmail(solicitud, accion) {
  let body = `
    <h2>Actualización de Solicitud de Canje</h2>
    <p>La solicitud #${solicitud.id} ha sido actualizada.</p>
    <p><strong>Nuevo Estado:</strong> ${accion}</p>
    <p><strong>Artículo:</strong> ${solicitud.articulo}</p>
    <p><strong>Solicitante:</strong> ${solicitud.email}</p>
  `;
  if (solicitud.comentarios) {
    body += `<p><strong>Comentarios:</strong> ${solicitud.comentarios}</p>`;
  }
  if (solicitud.articuloAlternativo) {
    body += `<p><strong>Artículo Alternativo Sugerido:</strong> ${solicitud.articuloAlternativo}</p>`;
  }
  body += `<p>Por favor, revisa el sistema para más detalles.</p>`;
  return body;
}

function enviarNotificacion(solicitudId, accion) {
  try {
    const solicitud = obtenerSolicitudPorId(solicitudId); // [cite: 27]
    if (!solicitud) {
      Logger.log("Error en notificación: No se encontró la solicitud " + solicitudId);
      return;
    }
    const destinatarios = obtenerDestinatarios(solicitud.email); // [cite: 27]
    MailApp.sendEmail({ // [cite: 28]
      to: destinatarios.to, // [cite: 28]
      cc: destinatarios.cc, // [cite: 28]
      subject: `[Canjes] Solicitud #${solicitud.id} - ${accion}`, // [cite: 28]
      htmlBody: generarCuerpoEmail(solicitud, accion) // [cite: 28]
    });
  } catch (error) {
    Logger.log("Error al enviar notificación para solicitud " + solicitudId + ": " + error.toString());
  }
}

function obtenerDestinatarios(emailSolicitante) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const hoja = ss.getSheetByName(CONFIG.hojaCorreos); // [cite: 29]
  return { // [cite: 30]
    to: emailSolicitante, // [cite: 30]
    cc: hoja.getRange("A2:A" + hoja.getLastRow()).getValues() // [cite: 30]
      .flat() // [cite: 30]
      .filter(email => email && email.trim() !== "") // [cite: 30]
      .join(",") // [cite: 30]
  };
}