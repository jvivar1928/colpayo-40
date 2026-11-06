# Colpayo 40 — Integraciones externas (setup)

Guía de las piezas que viven FUERA del `index.html`: Apps Script de Google y reglas extra de Firestore.
La app es el `index.html` de este repo (rama `main`, publicada con GitHub Pages).

## 1. Reglas extra de Firestore

Dentro de `match /databases/{database}/documents { … }` agregar:

```
match /publico/{docId} {
  allow read: if true;
  allow write: if request.auth != null && request.auth.uid == "Ea0OLqZbhIVm4sgPHMejV5s2dk22";
}
```

`publico/airbnb` guarda la URL del puente de Airbnb.

## 2. Puente de Airbnb (Google Apps Script)

**Qué hace:** lee el calendario de Airbnb (solo fechas) y los mails de confirmación de Airbnb del Gmail donde corre el script (de ahí saca el nombre del titular y la cantidad de huéspedes), y los cruza por el código de reserva. A la app le pasa SOLO: entrada, salida, nombre del titular y cantidad de huéspedes (adultos + niños; bebés y mascotas no cuentan). Nada de importes, teléfonos ni códigos.

**Por qué un puente:** el navegador no deja que la app lea el `.ics` de Airbnb directamente (CORS), el link tiene un token privado que no puede ir en el `index.html` (el repo de GitHub Pages es público), y el calendario de Airbnb no trae nombres ni huéspedes (Airbnb los sacó del iCal), así que salen de los mails.

**Requisito:** crear el script con la cuenta de Google cuyo Gmail recibe los mails de reservas de Airbnb.

**Pasos (primera vez):**
1. https://script.google.com → **Nuevo proyecto**. Nombre: `Puente Airbnb Colpayo`.
2. Borrar lo que haya y pegar el código de abajo. En la línea `const ICAL_URL = '';` pegar el link de Airbnb entre las dos comillas (Airbnb → Calendario → Disponibilidad → Conectar calendarios → Exportar calendario). Guardar.
3. En el menú de funciones de arriba (el que dice `doGet`) elegir `probar` y tocar **Ejecutar** (`doGet` no escribe nada en el registro). Autorizar (cuenta → Configuración avanzada → Ir a Puente Airbnb → Permitir; pide leer Gmail y conectarse a servicios externos). En el **Registro de ejecución** aparece una línea por reserva: revisar nombres y huéspedes. Si sale el error «No se pudo leer el calendario de Airbnb», el link de `ICAL_URL` está mal.
4. **Implementar → Nueva implementación** → engranaje → **Aplicación web**. "Ejecutar como": **Yo**. "Quién tiene acceso": **Cualquier usuario** (viene marcado "Solo yo"; con eso la app dice "No se pudo leer Airbnb"). Implementar.
5. Copiar la **URL de la aplicación web** (termina en `/exec`) y pegarla en la app: Calendario → tarjeta "Reservas de Airbnb" → Conexión con Airbnb → Guardar.

**Si se cambia el código:** Implementar → Gestionar implementaciones → lápiz → Versión: **Nueva versión** → Implementar. La URL no cambia, pero sin este paso sigue usando el código viejo. El puente guarda el resultado 10 minutos; `limpiarCache` lo fuerza.

**Para probar la URL:** abrirla en una ventana de incógnito; tiene que mostrar un texto que empieza con `{"actualizado":`.

Privacidad: cualquiera que tenga la URL `/exec` ve fechas, nombres de titulares y cantidad de huéspedes (lo mismo que ya ve cualquiera con el link de la app en modo visión).

```javascript
// Puente Airbnb → app Colpayo 40 (versión 4)
// Lee el calendario de Airbnb (fechas) y los mails de confirmación de Airbnb de este Gmail
// (nombre del titular y cantidad de huéspedes). A la app SOLO le pasa:
//   - fecha de entrada y de salida
//   - nombre del titular
//   - cantidad de huéspedes (adultos + niños; bebés y mascotas no cuentan)
// No manda importes, teléfonos, códigos ni nada más.

const ICAL_URL = ''; // ← pegá el link de Airbnb entre las dos comillas

function doGet() {
  const cache = CacheService.getScriptCache();
  const enCache = cache.get('airbnb_v4');
  if (enCache) return salida_(enCache);
  const data = JSON.stringify({ actualizado: new Date().toISOString(), eventos: armarEventos_() });
  cache.put('airbnb_v4', data, 600); // 10 minutos
  return salida_(data);
}

// Ejecutá esta función desde el editor (botón "Ejecutar") para ver qué encontró, en el registro.
function probar() {
  armarEventos_().forEach(function (e) { Logger.log(JSON.stringify(e)); });
}

// Si cambiaste algo y querés ver el resultado ya, sin esperar los 10 minutos.
function limpiarCache() {
  CacheService.getScriptCache().remove('airbnb_v4');
}

function salida_(json) {
  return ContentService.createTextOutput(json).setMimeType(ContentService.MimeType.JSON);
}

function armarEventos_() {
  const resp = UrlFetchApp.fetch(ICAL_URL, { muteHttpExceptions: true });
  const txt = resp.getContentText();
  // Si no se pudo leer el calendario, avisa con un error (y no guarda nada) en vez de mandar una lista vacía.
  if (resp.getResponseCode() !== 200 || txt.indexOf('BEGIN:VCALENDAR') < 0) {
    throw new Error('No se pudo leer el calendario de Airbnb. Revisá el link de ICAL_URL.');
  }
  const reservas = parsearIcal_(txt).filter(function (e) { return e.tipo === 'reserva'; });
  const desde = fechaISO_(new Date(Date.now() - 31 * 86400000));
  return reservas.map(function (e) {
    const out = { inicio: e.inicio, fin: e.fin };
    if (e.codigo && e.fin >= desde) {
      const datos = datosDelMail_(e.codigo);
      if (datos.nombre) out.nombre = datos.nombre;
      if (datos.huespedes) out.huespedes = datos.huespedes;
    }
    return out;
  });
}

function parsearIcal_(txt) {
  const lineas = txt.replace(/\r?\n[ \t]/g, '').split(/\r?\n/);
  const eventos = [];
  let ev = null;
  lineas.forEach(function (l) {
    if (l === 'BEGIN:VEVENT') ev = {};
    else if (l === 'END:VEVENT') { if (ev && ev.inicio && ev.fin) eventos.push(ev); ev = null; }
    else if (ev) {
      const i = l.indexOf(':');
      if (i < 0) return;
      const clave = l.slice(0, i).split(';')[0];
      const valor = l.slice(i + 1);
      if (clave === 'DTSTART') ev.inicio = fechaIcal_(valor);
      else if (clave === 'DTEND') ev.fin = fechaIcal_(valor);
      else if (clave === 'SUMMARY') ev.tipo = /reserv/i.test(valor) ? 'reserva' : 'bloqueo';
      else if (clave === 'DESCRIPTION') {
        const m = valor.match(/\b(HM[A-Z0-9]{6,})\b/);
        if (m) ev.codigo = m[1];
      }
    }
  });
  return eventos;
}

// Busca en Gmail el mail de confirmación de la reserva (por su código):
// del asunto saca el nombre y del cuerpo, la cantidad de huéspedes.
function datosDelMail_(codigo) {
  const hilos = GmailApp.search('from:airbnb.com "' + codigo + '"', 0, 10);
  for (let i = 0; i < hilos.length; i++) {
    const mensajes = hilos[i].getMessages();
    for (let j = 0; j < mensajes.length; j++) {
      const nombre = nombreDeAsunto_(mensajes[j].getSubject() || '');
      if (nombre) return { nombre: nombre, huespedes: contarHuespedes_(mensajes[j].getPlainBody() || '') };
    }
  }
  return {};
}

// "Reserva confirmada: Ana García llega el 2 nov."  /  "Reservation confirmed - Ana García arrives Nov 2"
function nombreDeAsunto_(asunto) {
  const m = asunto.match(/confirm(?:ada|ado|ed)\W+(.+?)\s+(?:llega|llegará|arrives|will arrive)\b/i);
  return m ? m[1].trim().slice(0, 80) : '';
}

// "Huéspedes: 2 adultos, 1 niño" → 3   /   "Guests 3 adults" → 3   (bebés y mascotas no cuentan)
function contarHuespedes_(texto) {
  const t = String(texto).replace(/\s+/g, ' ');
  const sumar = function (s) {
    const re = /(\d+)\s*(adultos?|adults?|niñ[oa]s?|child(?:ren)?)\b/gi;
    let total = 0, m;
    while ((m = re.exec(s))) total += Number(m[1]);
    return total;
  };
  const rotulo = /(?:hu[eé]spedes|viajeros|guests)\s*:?\s*(.{0,60})/gi;
  let m;
  while ((m = rotulo.exec(t))) {
    const n = sumar(m[1]);
    if (n) return n;
  }
  const m2 = t.match(/(\d+)\s*(?:hu[eé]spedes|viajeros|guests)\b/i);
  if (m2) return Number(m2[1]);
  return sumar(t);
}

function fechaIcal_(v) {
  const d = v.slice(0, 8);
  return d.slice(0, 4) + '-' + d.slice(4, 6) + '-' + d.slice(6, 8);
}

function fechaISO_(d) {
  return Utilities.formatDate(d, Session.getScriptTimeZone(), 'yyyy-MM-dd');
}
```

## 3. Saldo en Google Sheets (descartado)

Se descartó en nov-2026: la app ya no publica el saldo para Google Sheets. El saldo se sigue viendo en la pestaña Rendición. Cuando el editor abre la app, se borra solo el documento `publico/saldo` que había quedado publicado.

## 4. GitHub

- Repo: https://github.com/jvivar1928/colpayo-40 (GitHub Pages). La cuenta de GitHub ya está vinculada a Claude.
- Para que Claude suba directo, la conversación tiene que arrancar con el repo `jvivar1928/colpayo-40` elegido como fuente (claude.ai/code → selector de repo). Si no, Claude no puede pushear y el deploy es manual.
- Preferencia: que Claude pushee directo a `main` (sin PR).
- Deploy manual: repo → **Add file → Upload files** → arrastrar `index.html` → **Commit changes**. Publica en 1-2 minutos; refrescar la app.

## 5. Notas de comportamiento

- Montos: la coma es el decimal. Si se toca el punto del teclado numérico se convierte en coma. No tipear puntos de miles.
- Reservas abre por defecto en el mes (o año) actual.
- Reservas: casilla "3 huéspedes" (campo `tresHuespedes` en `reservas`); en el calendario sale "3 HUÉSP." el día de entrada y "3 HUÉSPEDES" en la leyenda. Al cargar desde Airbnb se marca sola si el mail dice 3 o más huéspedes.
- Airbnb: el botón **Cargar** de la tarjeta guarda la reserva directo (nombre, fechas y la casilla de 3 huéspedes), **sin pago ni limpieza**: pagan en dólares y los pesos se cargan después, editando la reserva. Mientras no tenga pago, en Reservas dice "Falta el pago" y en la tarjeta de Airbnb "Sin pago".
- Si se corta la conexión con Airbnb (Airbnb o Google caídos), la app sigue mostrando lo último que leyó (queda guardado en cada dispositivo) y la tarjeta dice "No se pudo leer Airbnb · último dato: hh:mm". Las reservas ya cargadas en la app no dependen de Airbnb. Si se borra la URL del puente a propósito, dejan de verse las de Airbnb que faltan cargar.
- Una reserva puede entrar el mismo día que sale otra (también las de Airbnb). Lo que no se puede es compartir una noche, ni que dos reservas empiecen el mismo día. En el calendario ese día se ve partido en diagonal: arriba a la izquierda el color del que sale y abajo a la derecha el del que entra, con su nombre (y "3 HUÉSP." si corresponde). Si el que entra es de Airbnb y falta cargarlo, esa mitad va en el color de Airbnb.
