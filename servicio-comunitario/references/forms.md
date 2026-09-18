# Formularios: registro y editar

Ambos comparten estructura, estilo y JS. `editar` = `registro` + precarga desde
BD + `UPDATE` en vez de `INSERT`.

## Patrón PRG (Post-Redirect-Get) con mensajes en sesión

Al enviar, se procesa en la misma página; ante error se guarda lo enviado y un
mensaje, y se **redirige** (evita reenvíos y conserva datos):

```php
$mensaje  = $_SESSION['mensaje_<pagina>'] ?? "";  unset($_SESSION['mensaje_<pagina>']);
$form_data = $_SESSION['form_data'] ?? [];         unset($_SESSION['form_data']);
// ... cargar catálogos con las funciones del DAO ...

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // leer $_POST, normalizar textos con mb_strtoupper(trim(...), 'UTF-8')
    // resolver condicionales a NULL cuando no aplican
    $errores = [];
    if (!$campo_id) $errores[] = 'el campo X';
    // ...
    if ($errores) {
        $_SESSION['form_data'] = $_POST;
        $_SESSION['mensaje_<pagina>'] = "<div class='alert alert-danger shadow-sm fw-bold'><i class='bi bi-exclamation-triangle-fill me-2'></i>Por favor complete: ".implode(', ', $errores).".</div>";
        header("Location: <pagina>.php"); exit;
    }
    // INSERT/UPDATE con prepared statement...
    $_SESSION['mensaje_<destino>'] = "<div class='alert alert-success ...'>...</div>";
    header("Location: <destino>.php"); exit;
}

// Helpers para repoblar el formulario:
function fd($k,$d=''){ global $form_data; return htmlspecialchars($form_data[$k] ?? $d); }
function fdsel($k,$v){ global $form_data; return (($form_data[$k] ?? '')==$v)?'selected':''; }
```

En `editar`, si NO viene de un error (form_data vacío), precarga `$form_data`
desde el registro para reutilizar `fd()`/`fdsel()`:

```php
$reg = obtenerRegistroPorId($pdo, $id);
if (!$reg) { $_SESSION['mensaje_historial_<modulo>']="..."; header('Location: <modulo>_historial.php'); exit; }
if (empty($form_data)) {
  $form_data = [
    'campo_id' => $reg['campo_id'],
    'en_maracaibo' => ((int)$reg['municipio_id'] === $maracaibo_id) ? 'SI' : 'NO',
    // ... mapear cada campo del form al valor de BD ...
  ];
}
```

## Estilo del formulario (conservar)

```html
<style>
  body { background: linear-gradient(90deg, rgba(210,0,90,1) 0%, rgba(22,67,119,1) 100%) !important; font-family: Arial, sans-serif; }
  .bg-gradient-custom { background: linear-gradient(90deg, rgba(210,0,90,1) 0%, rgba(22,67,119,1) 100%) !important; }
  .card-form { border-radius: 12px; border: none; box-shadow: 0 4px 12px rgba(0,0,0,0.08); }
  .seccion-titulo { border-bottom: 2px solid #d2005a; padding-bottom: 5px; color: #d2005a; font-weight: bold; margin-top: 25px; margin-bottom: 15px; }
  .bloque-seccion { background-color: #f8f9fa; border-left: 4px solid #d2005a; padding: 15px; margin-bottom: 15px; border-radius: 0 8px 8px 0; }
</style>
```

## Placeholder de los <select> (conserva la selección tras error)

```html
<option value="" disabled <?= empty($form_data['<campo>']) ? 'selected' : '' ?>>Seleccione...</option>
```
`disabled` impide re-elegir el vacío; el `selected` condicional solo aplica si
no hay valor previo (así, tras un error, queda seleccionado lo que el usuario
eligió, no el placeholder).

## Etiquetas de campos obligatorios

Añade `<span class="text-danger">*</span>` a cada etiqueta obligatoria y una
leyenda: *"Los campos marcados con * son obligatorios."*

## Validación en el navegador (clave: no usar `required` nativo)

El `required` nativo **bloquea el envío antes de que corra el PHP**, y si un
campo obligatorio está oculto (condicional), el navegador falla en silencio. Por
eso el `<form>` lleva `novalidate` + `onsubmit`, y una función lista los
faltantes en un contenedor `#alerta_js` y los marca en rojo:

```html
<div id="alerta_js"></div>
<form ... id="formRegistro" onsubmit="return validarFormulario(event);" novalidate>
```

```js
function nombreSel(id){const s=document.getElementById(id);return s&&s.selectedIndex>=0?(s.options[s.selectedIndex].dataset.nombre||''):'';}

function validarFormulario(event){
  const faltantes=[]; let primero=null;
  document.querySelectorAll('.form-control,.form-select').forEach(el=>el.classList.remove('is-invalid'));
  function check(id,etiqueta,aplica){
    if(aplica===false) return;
    const el=document.getElementById(id); if(!el) return;
    if(!(el.value||'').trim()){ el.classList.add('is-invalid'); faltantes.push(etiqueta); if(!primero) primero=el; }
  }
  // check('campo_id','Etiqueta');
  // check('condicional','Etiqueta', <condición JS booleana>);
  const cont=document.getElementById('alerta_js');
  if(faltantes.length){
    cont.innerHTML="<div class='alert alert-danger shadow-sm fw-bold'><i class='bi bi-exclamation-triangle-fill me-2'></i>Debe completar los siguientes campos obligatorios: "+faltantes.join(', ')+".</div>";
    if(event&&event.preventDefault) event.preventDefault();
    if(primero) primero.focus();
    cont.scrollIntoView({behavior:'smooth',block:'center'});
    return false;
  }
  cont.innerHTML=''; return true;
}
```
Mantén también la validación en el **servidor** como respaldo (arriba, en el
bloque POST): el JS puede saltarse.

## Campos condicionales (mostrar/ocultar)

Para cada `select` o radio que controla otros campos, una función `toggle*` que
muestra el `wrapper` y activa/limpia su input. Guarda el nombre de la opción en
`data-nombre` para comparar por texto:

```html
<option value="<?= $x['id'] ?>" data-nombre="<?= htmlspecialchars($x['nombre']) ?>" <?= fdsel('campo_id',$x['id']) ?>><?= htmlspecialchars($x['nombre']) ?></option>
```

```js
function toggleX(){
  const nombre = xSelect.options[xSelect.selectedIndex]?.dataset.nombre || '';
  wrapY.style.display = (nombre === 'OTRO') ? 'block' : 'none';
  if (nombre !== 'OTRO') document.getElementById('y').value = '';
}
xSelect.addEventListener('change', toggleX);
```

Reúne todos los toggles en `refrescarCondicionales()` y llámala **al cargar**
(reproduce el estado tras un error PRG) y **en el reset** del formulario. El
reset ocurre DESPUÉS del evento, por eso el `setTimeout`:

```js
function refrescarCondicionales(){ toggleX(); toggleY(); /* ... */ }
document.getElementById('formRegistro').addEventListener('reset', function(){
  setTimeout(function(){
    refrescarCondicionales();
    document.querySelectorAll('.form-control,.form-select').forEach(el=>el.classList.remove('is-invalid'));
    document.getElementById('alerta_js').innerHTML='';
  }, 0);
});
refrescarCondicionales();
```

## Ubicación Maracaibo / Foráneo → municipio_id + parroquia_id

Patrón usado en varios módulos: un combo "¿en Maracaibo?" (SI/NO) decide.
- SI → `municipio_id = <id Maracaibo>`, se muestra y exige `parroquia_id`.
- NO → `municipio_id = <foráneo elegido>`, `parroquia_id = NULL`.

El id de Maracaibo se obtiene por nombre en PHP y se detecta en JS vía un
`data-maracaibo` en el select, o mapeando el municipio del registro.
