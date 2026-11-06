# Colpayo 40 — notas para Claude

App de gestión del alquiler temporario de Lucy en Caballito (Colpayo 40). Todo vive en `index.html`
(HTML + CSS + JS en un `<script type="module">`), publicado con GitHub Pages desde `main`:
https://jvivar1928.github.io/colpayo-40/

## Convenciones
- Responder en español rioplatense. Explicar con detalle las acciones (qué tocar y dónde), más que los temas técnicos.
- Antes de entregar: validar que `<div` y `</div>` estén balanceados y correr `node --check` sobre el script del módulo.
- Decir siempre qué hay que subir o hacer fuera del repo (Firebase, Apps Script, etc.).
- Pushear directo a `main` (sin PR). GitHub Pages publica en 1-2 minutos.
- El repo es público: nunca poner links privados (como el del calendario de Airbnb) ni claves.
- GitHub Pages procesa los `.md` con Jekyll: no escribir en ellos llaves dobles ni llave seguida de porcentaje (es sintaxis de Liquid y rompe la publicación).

## Cómo está armada
- Firebase, proyecto `colpayo40-223e6`: Auth (el editor entra con correo; el resto ve en modo visión, anónimo)
  y Firestore con `reservas`, `gastos`, `notas`, `mesesCerrados`, `conversionesUSD`, `rendicionesUSD` y `publico`
  (solo `publico/airbnb`, con la URL del puente).
- Editor: constante `EDITOR_UID` en `index.html`.
- Montos en pesos, con coma decimal.
- Airbnb: un puente en Google Apps Script lee el calendario de Airbnb y los mails de confirmación, y le pasa a la app
  fechas, nombre del titular y cantidad de huéspedes. Reglas de Firestore, pasos y código del puente:
  `docs/setup-integraciones.md`. El código del puente vive en Apps Script; si se cambia, actualizar también esa guía.
