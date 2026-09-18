# Historial y exportaciones

## Historial (listado)

- Consulta con **JOINs** a los catálogos para mostrar `nombre`, no ids.
- Buscador en vivo (submit automático con debounce de 800 ms) y botón limpiar.
- Acciones por fila: Editar, Descargar PDF, Eliminar (con `confirm()`).
- Botones de cabecera: Exportar PDF, Exportar Excel, Nueva.

### Consulta con búsqueda (placeholders ÚNICOS)

Con `ATTR_EMULATE_PREPARES=false` **no se puede reutilizar** un placeholder.
Usa `:b1..:bN`, todos con el mismo valor:

```php
$sql = "SELECT r.*, cat.nombre AS cat_nombre /* ...otros JOIN... */
        FROM <modulo>_registros r
        LEFT JOIN <catalogo> cat ON cat.id = r.cat_id
        WHERE 1=1";
$params = [];
if ($buscar !== "") {
    $sql .= " AND (r.nombre_actividad LIKE :b1 OR cat.nombre LIKE :b2 OR r.sector LIKE :b3)";
    $like = "%$buscar%";
    foreach (['b1','b2','b3'] as $p) $params[':'.$p] = $like;
}
$sql .= " ORDER BY r.id DESC";
$stmt = $pdo->prepare($sql); $stmt->execute($params);
$registros = $stmt->fetchAll(PDO::FETCH_ASSOC);
```

### Buscador en vivo (JS)

```js
let t;
document.getElementById('tablaBuscador').addEventListener('input', function(){
  clearTimeout(t); t=setTimeout(()=>document.getElementById('formBuscador').submit(), 800);
});
function exportarPDF(){ const b=document.getElementById('tablaBuscador').value; window.open('<modulo>_pdf.php'+(b?'?buscar='+encodeURIComponent(b):''),'_blank'); }
function exportarExcel(){ const b=document.getElementById('tablaBuscador').value; window.open('<modulo>_exportar_excel.php'+(b?'?buscar='+encodeURIComponent(b):''),'_blank'); }
```

Las exportaciones **respetan el filtro activo** pasando `?buscar=`. Mensajes del
historial (borrado/edición) se leen desde `$_SESSION['mensaje_historial_<modulo>']`.

## Eliminar (mcbo/<modulo>_eliminar.php)

Valida el id, verifica existencia, `DELETE` con prepared statement, y redirige
al historial con un mensaje en `$_SESSION['mensaje_historial_<modulo>']`. No es
destructivo irreversible más allá de lo esperado, pero mantén el `confirm()` en
el botón.

## Exportar Excel (sin librería)

Un HTML `<table>` servido como `.xls` con BOM UTF-8 (Excel respeta tildes/ñ):

```php
header("Content-Type: application/vnd.ms-excel; charset=utf-8");
header("Content-Disposition: attachment; filename=\"Reporte_<Modulo>_".date('Y-m-d_H-i').".xls\"");
header("Pragma: no-cache"); header("Expires: 0");
echo "\xEF\xBB\xBF"; // BOM
```
Encabezado con banda `#D2005A`, `th` azul `#164377`. Filtra con la misma
consulta+JOINs del historial (respetando `?buscar=`).

## Exportar PDF (listado) — sin librería

Página HTML con estilo de impresión y `window.print()` automático:

```php
// @page { size: letter landscape; margin: 12mm 10mm; }
// body { ... padding: 6mm; }   <- padding para que se vea todo el contenedor
```
```html
<script>window.onload=function(){ if(!location.search.includes('noprint')) setTimeout(()=>window.print(),500); };</script>
```
Incrusta el logo en base64 para que aparezca al imprimir:

```php
$logoBase64=''; $p='imagenes/alcaldia-maracaibo.png';
if(file_exists($p)) $logoBase64='data:image/'.pathinfo($p,PATHINFO_EXTENSION).';base64,'.base64_encode(file_get_contents($p));
```

## Descargar PDF individual (planilla por registro)

`<modulo>_descargar_pdf.php?id=` — una planilla A4 vertical con secciones
(`.section-header` azul, `.table-data` etiqueta/valor, badges Sí/No, área de
firma). Usa el banner `imagenes/alcaldia_logo_encabezado.png` a todo el ancho y
`body { padding: 8mm; }`. Botón "Imprimir / Guardar PDF" (`.no-print`) +
`window.print()` automático. Consulta el registro por id con los JOINs a
catálogos.

**Importante (padding)**: los originales usan `body{padding:0}` y dependen solo
del `@page margin`; añade un `padding` al body (6–8mm) para que el contenedor
se vea completo tanto en pantalla como al imprimir. Mantén el resto del estilo
idéntico al original del módulo gemelo.
