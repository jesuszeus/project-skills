# Extraer la información de la fuente (cualquier formato)

La fuente que define un módulo **no siempre es un formulario de KoboToolbox**.
Puede ser un PDF, un documento de texto o Word, fotos/escaneos de una planilla
de papel, un Google Forms, una hoja de cálculo con respuestas, o incluso una
descripción verbal del usuario. Sea cual sea, tu meta es siempre la misma:

**Derivar el modelo de datos**:
1. La lista de **campos** (con su etiqueta legible).
2. El **tipo** de cada uno (texto, número, fecha, sí/no, selección).
3. Para cada selección, sus **opciones** (que serán un catálogo).
4. Qué campos son **obligatorios** y qué campos son **condicionales** (aparecen
   solo cuando otro campo toma cierto valor).

Con eso construyes el SQL (`references/database.md`) y las páginas.

## Regla transversal: confirma antes de construir

Salvo Kobo (que trae el modelo explícito), casi todas las fuentes son
**ambiguas**: un PDF o una foto no dicen qué campo es obligatorio ni cuál es un
catálogo. Por eso, **extrae un borrador del modelo y valídalo con el usuario**
(lista de campos + tipos + opciones + reglas) antes de escribir tablas. Es
barato preguntar y caro rehacer el esquema.

## Por tipo de fuente

### KoboToolbox / Enketo (URL de formulario)

Es el caso ideal: el modelo está en el DOM. Abre la URL en el navegador y usa
JS para volcar campos, opciones y reglas `relevant`.

```js
// Campos ordenados
const out=[];
document.querySelectorAll('.question').forEach(q=>{
  const n=q.querySelector('[name*="/"]'); if(!n) return;
  out.push({ path:n.getAttribute('name').split('/').slice(-2).join('/'),
    label:(q.querySelector('.question-label')?.innerText||'').trim().split('\n')[0],
    tipo:n.getAttribute('data-type-xml')||n.type||n.tagName });
});
JSON.stringify(out,null,1);
```
```js
// Opciones de cada select
const o={};
document.querySelectorAll('select').forEach(s=>{ o[s.getAttribute('name')]=[...s.options].map(x=>x.value+' | '+x.text); });
JSON.stringify(o,null,1);
```
Para las reglas condicionales, fija un valor y observa qué queda visible
(`setSel` + comprobar `offsetParent`), porque las expresiones `relevant` viven
en contenedores envolventes y emparejarlas a mano es frágil.

### PDF

Léelo con la herramienta `Read` (admite el rango de páginas) o, para PDFs
complejos con tablas/escaneos, apóyate en la skill `pdf`. Busca:
- Etiquetas de campo (a veces seguidas de "____" o casillas).
- Listas de opciones (viñetas, casillas, tablas "Sí/No").
- Secciones que agrupan campos (se vuelven bloques del formulario).
Si el PDF es escaneado (imagen), trátalo como imagen (abajo) u OCR con la skill
`pdf`.

### Imágenes / fotos / escaneos de planillas

Puedes **leer la imagen directamente** (visión): transcribe los campos y sus
opciones. Cuida:
- Texto borroso o manuscrito: transcribe lo que puedas y **pide al usuario que
  confirme** los términos dudosos, sobre todo nombres propios y siglas.
- Casillas marcadas: indican opciones de un select o booleanos.
- Varias fotos: pídelas todas para no perder secciones.

### Google Forms

Abre la URL de respuesta (`.../viewform`) en el navegador y extrae preguntas y
opciones del DOM. Las preguntas suelen estar en `[role="listitem"]`:

```js
[...document.querySelectorAll('[role="listitem"]')].map(it=>({
  pregunta: it.querySelector('[role="heading"]')?.innerText?.trim(),
  opciones: [...it.querySelectorAll('[role="radio"],[role="checkbox"]')].map(o=>o.getAttribute('aria-label')).filter(Boolean)
})).filter(x=>x.pregunta);
```
Si el usuario tiene el enlace de **edición**, también sirve para ver la
estructura. Los tipos (párrafo, opción múltiple, casillas, desplegable) mapean
directo a texto / select_one / select_multiple.

### Documentos de texto o Word (.docx/.txt/.md)

Léelos con `Read` (o la skill `docx` para Word con formato). Normalmente traen
una lista estructurada de campos; conviértela al modelo. Ojo con listas
numeradas que en realidad son opciones de un mismo campo.

### Hoja de cálculo / CSV con respuestas existentes

Si te dan datos ya recolectados (Excel/CSV), cada **columna** es un campo y sus
**valores distintos** insinúan el catálogo. Usa la skill `xlsx` para leer.
Útil además para **migrar** datos históricos al nuevo esquema.

### Descripción verbal del usuario

A veces solo hay una conversación ("necesito registrar actividades con fecha,
responsable, tipo…"). Toma nota, propón el modelo y confírmalo. Trátalo igual
que cualquier otra fuente: campos, tipos, opciones, obligatorios, condicionales.

## Mapear al esquema (igual para todas las fuentes)

- Selección de una opción → catálogo (id+nombre) + columna `X_id` (FK).
- Selección con "Otro" → columna extra `X_otro` (texto) visible solo con Otro.
- Sí/No → `tinyint` booleano.
- Número → `int`; texto → `varchar`/`text`; fecha → `date`.
- Campo condicional (`aparece si A = valor`) → columna **nullable**, obligatoria
  en la app solo cuando se cumple la condición.
- Selección múltiple → tabla puente (registro_id, opcion_id) **o**, si el módulo
  gemelo lo hace así, un campo JSON; sigue el patrón del proyecto existente.

## Cuidado con bugs y vacíos de la fuente

Ninguna fuente es infalible: un Kobo puede traer una regla imposible, un PDF
puede omitir marcar qué es obligatorio, una foto puede cortar una sección.
Implementa la **intención** del formulario, no el error literal, y **avísale al
usuario** de cualquier corrección o supuesto que hayas tomado.
