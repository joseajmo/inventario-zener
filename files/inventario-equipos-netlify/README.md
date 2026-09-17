# Inventario de Equipos — código fuente

Aplicación web progresiva (PWA) de un solo archivo principal, sin frameworks ni proceso de compilación. Se edita directamente y se vuelve a desplegar.

## Archivos

- `index.html` — toda la app: HTML + CSS + JavaScript en un único archivo (para mantenerlo simple de editar y desplegar).
- `manifest.json` — metadatos de la PWA (nombre, icono, colores). Edítalo si cambias el nombre o el color de marca.
- `sw.js` — service worker: cachea la app para que funcione sin conexión. Si cambias el número de versión de caché (`CACHE_NAME`), fuerzas a los móviles a descargar la versión nueva.
- `icons/` — iconos de la app (192px, 512px, 512px "maskable" para Android). Generados con Pillow; si quieres otro diseño, sustitúyelos manteniendo el mismo tamaño y nombre de archivo.
- `assets/` — todas las librerías de terceros, alojadas como archivos locales (ver más abajo por qué) más el logo de Zener.

**Todas las librerías van alojadas localmente en `assets/`, ninguna se carga desde un CDN externo** (XLSX/SheetJS, jsQR, EmailJS, ExcelJS, generador de QR). El único servicio externo real que sigue usando la app es el propio login de Google (`accounts.google.com`), porque por su naturaleza tiene que hablar con los servidores de Google. Esto se cambió después de detectar que, en algunas redes móviles, las librerías cargadas desde CDNs (cdnjs, jsdelivr) no llegaban a descargarse —posiblemente bloqueadas por el operador o alguna app de seguridad del teléfono—, lo que rompía silenciosamente la lectura de QR y la generación de Excel. Si en el futuro actualizas alguna de estas librerías, descárgala de nuevo desde npm (`npm pack <paquete>`) y sustituye el archivo correspondiente en `assets/`, sin volver a apuntar a un CDN.

- `assets/logo-zener.png` — logotipo de Zener, usado en la cabecera y en cada formulario. Para cambiarlo, sustituye este archivo manteniendo el nombre (o actualiza las etiquetas `<img src="assets/logo-zener.png">` en `index.html` si cambias el nombre).
- `assets/exceljs.min.js` — librería ExcelJS (build oficial de su npm, sin modificar) usada para **generar** el Excel del inventario con imágenes incrustadas.
- `assets/qrcode.min.js` — bundle propio (generado con esbuild a partir del paquete npm `qrcode`) que convierte texto en una imagen QR, usado solo para regenerar el QR dentro del Excel.
- `assets/jsQR.js` — librería jsQR (build oficial de su npm), usada para leer/decodificar códigos QR desde la cámara o desde una foto.
- `assets/xlsx.full.min.js` — librería SheetJS/XLSX (build oficial de su npm), usada solo para **leer** el Excel del catálogo al importarlo (no para generar el Excel del inventario, eso lo hace ExcelJS).
- `assets/email.min.js` — SDK de EmailJS (build oficial de su npm), usado para la alternativa de envío sin Google.

No hay `npm install` ni build: es HTML plano. Para probarlo en local basta con:

```bash
cd app
python3 -m http.server 8000
# abrir http://localhost:8000 en el navegador
```

(La cámara/QR y el Service Worker solo funcionan sobre HTTPS o `localhost`, nunca abriendo el archivo con doble clic.)

- El **Google Client ID** de Zener ya está incrustado por defecto en el código (`ajustes.googleClientId` en `index.html`), así que los empleados no tienen que rellenarlo — solo pulsan "Iniciar sesión con Google" y usan su propio Gmail. Si algún día cambias de dominio (otra URL de Netlify, dominio propio, etc.), hay que crear un nuevo Client ID en Google Cloud Console con ese origen autorizado y sustituir el valor en esta misma línea.

## Estructura del JavaScript (dentro de `index.html`)

Todo el script vive en un único IIFE `(function(){ ... })()` al final del archivo, organizado en bloques con comentarios `/* ---- nombre ---- */`:

1. **Almacenamiento** (`loadJSON`/`saveJSON`) — todo se guarda en `localStorage` del navegador, por dispositivo. Claves: `inv_equipos_v2`, `inv_catalogo_v2`, `inv_ajustes_v2`.
2. **Alta inicial obligatoria** (`checkOnboarding`) — pantalla que pide nombre/empresa/provincia/clave/fecha/correos la primera vez.
3. **Google Sign-In + envío por Gmail** — cliente OAuth (`ensureGoogleTokenClient`), construcción del email MIME (`buildMimeMessage`) y envío real vía Gmail API (`intentarEnvioGmail`).
4. **Navegación** — cuatro vistas (Inventario / Añadir / Catálogo / Ajustes) mostradas/ocultadas con la clase `.active`.
5. **Inventario** (`renderInventario`, `rowHTML`, `openDetalle`) — listado, búsqueda y ficha de cada equipo.
6. **Formulario añadir/editar** (`openForm`, guardado en `btnGuardarEquipo`) — incluye foto (redimensionada a JPEG con `resizeImageToDataURL`), cantidad, estado, y el QR completo si se escaneó.
7. **Escáner QR** (`startScan`, `onQRDetected`, `decodificarQRDesdeArchivo`) — usa la cámara vía `getUserMedia` + librería `jsQR`. Lee los primeros `LONGITUD_CODIGO_QR` (8) caracteres como código de producto y busca la descripción en el catálogo. **Importante:** la cámara en directo solo funciona sobre HTTPS (o `localhost`) — si se abre el sitio por `http://`, el navegador bloquea el acceso a la cámara y la app lo avisa explícitamente. Por eso hay un segundo método, "Subir foto del QR" (`fotoQRInput`), que usa un simple `<input type="file" capture="environment">` y decodifica la imagen estática con `jsQR`: funciona en muchos más navegadores/contextos que la cámara en directo y es la alternativa recomendada si el escaneo en vivo falla en algún dispositivo.
8. **Catálogo** (`mapCatalogRow`, importación con `XLSX`) — descarga manual del Excel + importación local para que la búsqueda sea instantánea y funcione sin conexión.
9. **Ajustes** (`renderAjustes`, guardado) — perfil del RRPP, credenciales opcionales de EmailJS, Client ID de Google.
10. **Envío del inventario** (`enviarInventario`) — intenta en este orden: Gmail (si hay sesión de Google) → EmailJS (si está configurado) → compartir con Web Share API (adjunta fotos) → `mailto:` como último recurso.
11. **Excel del inventario** (`construirExcelInventario`, ahora asíncrona) — genera un `.xlsx` con dos hojas: "Inventario" (todas las líneas, con el QR original **incrustado como imagen escaneable** en su propia columna cuando el equipo se dio de alta escaneando un QR) y "Datos RRPP" (nombre, empresa, provincia, clave, fechas, correos). Se adjunta automáticamente al enviar por Gmail o al compartir, y también se puede descargar suelto con el botón "Descargar copia en Excel". EmailJS no soporta adjuntar archivos de forma programática (limitación de su SDK), así que por esa vía solo se envía el resumen en el cuerpo del correo.
   - La generación del Excel usa **ExcelJS** (`assets/exceljs.min.js`), no SheetJS/XLSX: la versión gratuita de SheetJS no permite escribir imágenes en las celdas, y para poder incrustar el QR era necesario ExcelJS. SheetJS (`XLSX`, por CDN) se sigue usando solo para **leer** el Excel del catálogo al importarlo.
   - El QR se regenera como imagen a partir del texto guardado en `qrCompleto` usando **`assets/qrcode.min.js`** (bundle propio, generado con esbuild a partir del paquete npm `qrcode`, porque esa librería no publica un build de navegador listo en un CDN). Escaneando esa imagen con la cámara del móvil se recupera exactamente el mismo texto que se leyó al dar de alta el equipo — lo comprobé extrayendo la imagen de un Excel generado y decodificándola con un lector de QR real, coincide carácter por carácter.
   - Si un equipo no tiene QR asociado (se añadió a mano), esa celda muestra un guion en vez de imagen.
12. **Historial** (`historial`, pestaña "Historial") — el botón "Finalizar inventario y empezar uno nuevo" (vista Inventario) archiva una copia completa del inventario actual (equipos + datos del RRPP de ese momento) en `inv_historial_v1` y vacía la lista de trabajo. Así el inventario nunca es acumulativo: cada ronda empieza limpia, pero todas las rondas anteriores quedan consultables y descargables (mismo Excel de dos hojas) desde la pestaña Historial, con opción de eliminarlas.
13. **Código/QR único por inventario** — un mismo código (leído por QR o escrito a mano) no se puede registrar dos veces dentro del inventario **actual** (se trata como identificador de una unidad física concreta). Si se reescanea un QR ya registrado, la app avisa y abre esa línea existente para editarla en vez de duplicarla (`onQRDetected`); si se intenta guardar a mano un código repetido, se bloquea el guardado con el mismo aviso (`btnGuardarEquipo`). Esta comprobación es por inventario de trabajo: al "Finalizar inventario y empezar uno nuevo" el mismo código vuelve a estar disponible, ya que pertenece a una ronda distinta.

## Modelo de datos de un equipo

```js
{
  id, descripcion, marca, modelo, codigo, cantidad,
  ubicacion, notas, estado,      // "bueno" | "regular" | "reparacion" | "baja"
  foto,                          // dataURL JPEG o null
  qrCompleto,                    // texto completo leído del QR, o null si se añadió a mano
  fechaAlta, fechaMod
}
```

## Cosas típicas que vas a querer cambiar

- **Colores/marca**: variables CSS al principio del `<style>` (`--primary`, `--accent`, etc.) y `theme_color`/`background_color` en `manifest.json`.
- **Campos del formulario de equipo**: sección `<section id="view-anadir">` en el HTML + el objeto `data` dentro del listener de `btnGuardarEquipo`.
- **Longitud del código leído por QR**: constante `LONGITUD_CODIGO_QR` (actualmente 8).
- **Columnas que acepta el Excel del catálogo**: función `mapCatalogRow` (ahora mismo intenta "código"/"descripción", con "nombre" como alias antiguo).
- **Plantilla del correo enviado**: funciones `construirCuerpoTexto` y `construirCuerpoHTML`.

## Publicar cambios

1. Edita `index.html` (o los otros archivos).
2. Vuelve a subir la carpeta entera a tu hosting (Netlify Drop, GitHub Pages, etc.), reemplazando los archivos existentes.
3. Los móviles que ya tengan la app instalada verán la actualización automáticamente la próxima vez que abran la app con conexión (gracias al Service Worker), normalmente en la segunda apertura.
