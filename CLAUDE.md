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
- Gastos, Resumen y Costeo tienen un desplegable de Año (arranca en el actual) que filtra el de Mes. En Rendición, el Año filtra
  las listas de conversiones y rendiciones (los saldos son siempre el total).
- Copia de seguridad e importación: tarjeta en Gastos (solo editor). La copia baja todo en JSON. Importar aplica un archivo
  `colpayo40-importacion` (operaciones set/update/delete con controles de saldo; hasta 500 cambios van en un solo lote, todo o nada)
  y antes baja una copia sola. Lo importado del Excel viejo (ago-2023 a jul-2026) tiene ids que empiezan con `xls-` y gastos sin categoría.
- Los datos privados (copias, archivos de importación) nunca van al repo: se le pasan al usuario como archivos.
- Meses cerrados: en reservas y gastos va un candado en lugar del lápiz y el tacho, y no se puede pasar algo de un mes cerrado
  a otro mes (para cambiarlo hay que reabrir el mes desde Resumen).
- Reservas sin pago (las de Airbnb se cargan así): botón "Cargar pago" para el editor; en modo visión o con el mes cerrado se
  muestra el aviso "Pago sin cargar". Al editar una reserva se conserva su comisión (las importadas no son todas del 30%).
- Cuotas de la tarjeta: van en la nota del gasto como "3/9". Los gastos con una barra en la nota tienen un botón que deja el
  formulario listo con la cuota siguiente, un mes después (no aparece en la última cuota ni si la siguiente ya está cargada);
  si la barra no es de una cuota, el botón es "Duplicar". Nunca guarda solo: hay que tocar "Agregar gasto".
- Buscadores en Reservas y Gastos (ignoran acentos y mayúsculas; buscan en todos los meses). En el celular, el formulario de
  Reservas arranca plegado detrás del botón "Agregar reserva".
- Aviso de conexión: `guardar()` envuelve cada escritura a Firestore y cuenta las que faltan confirmar; el cartel de abajo avisa
  si no hay conexión o si el guardado tarda. Toda escritura nueva tiene que pasar por `guardar()`.
- Airbnb: un puente en Google Apps Script lee el calendario de Airbnb y los mails de confirmación, y le pasa a la app
  fechas, nombre del titular y cantidad de huéspedes. Reglas de Firestore, pasos y código del puente:
  `docs/setup-integraciones.md`. El código del puente vive en Apps Script; si se cambia, actualizar también esa guía.
