---
name: servicio-comunitario
description: >-
  Construye y extiende módulos web PHP para los sistemas de la Alcaldía de
  Maracaibo del servicio comunitario de Jesús (proyectos mcbo-protocolo y
  maracaibo-gaita): login multi-módulo, CRUD (registro/historial/editar/
  eliminar), exportación PDF/Excel y estadísticas con Chart.js, a partir de
  cualquier fuente (KoboToolbox, PDF, imágenes o fotos de planillas, Google
  Forms, texto/Word, hoja de cálculo, o una descripción del usuario). USA ESTE
  SKILL siempre que trabajes en esos repos o en cualquiera con la misma
  arquitectura (index.php multi-módulo + validacion.php + usuarios_sistema con
  una columna de permiso por módulo + PDO en includes/dbconnection.php + páginas
  PHP con prefijo por módulo), o cuando pida: crear un módulo, agregar
  registro/historial/estadísticas, convertir un formulario o documento a
  PHP+MySQL, o replicar el estilo de un módulo existente. Aplícalo aunque no
  nombre "skill" si el contexto es uno de estos proyectos de servicio comunitario.
---

# Servicio Comunitario — Módulos PHP Alcaldía de Maracaibo

Estos proyectos son sistemas internos de gestión para direcciones de la
Alcaldía de Maracaibo. Cada "proyecto" (repo) es en realidad una plataforma
**multi-módulo**: un mismo login sirve a varios módulos y cada módulo es un
CRUD que **digitaliza un formulario de campo**. Ese formulario puede llegar en
cualquier formato: KoboToolbox, PDF, fotos/escaneos de una planilla, Google
Forms, un documento de texto/Word, una hoja de cálculo, o solo la descripción
del usuario. Sea cual sea, el primer paso es derivar el modelo de datos de esa
fuente (ver `references/data-sources.md`).

El objetivo de este skill es que puedas **crear un módulo nuevo completo o
extender uno existente en minutos**, reutilizando la arquitectura y el estilo
ya probados, sin reinventar decisiones ya tomadas. La regla de oro es:
**conservar el estilo y la estructura existentes, adaptar solo los datos** al
modelo del módulo nuevo.

## Regla de trabajo primero

1. **Lee antes de escribir.** El usuario suele editar archivos en disco entre
   turnos (reordena campos, ajusta layout). Relee el archivo objetivo y sus
   hermanos del módulo gemelo (eventos, gaita, protocolo) antes de editar. No
   reescribas de cero lo que puedes adaptar.
2. **Parte de un módulo existente como plantilla.** Para cada página nueva,
   busca su equivalente (`mcbo_eventos_registro.php`, `maracaibo_gaita_*`,
   `mcbo_protocolos_*`) y consérvala: mismo `<style>`, misma estructura, mismo
   patrón PHP. Cambia solo campos, tablas y textos.
3. **Verifica sintaxis** tras cada archivo PHP: `php -l archivo.php`.
4. **No inventes datos ni columnas.** Los campos salen del formulario Kobo o
   del modelo SQL vigente.

## Arquitectura (cómo encaja todo)

```
index.php            Selector de módulo + login. $config_modulos define cada
                     módulo (titulo, sub, color, btn_color). Redirige a
                     <modulo>_dashboard.php según la sesión.
validacion.php       Procesa el login: SELECT usuarios_sistema WHERE usuario;
                     compara password (texto plano, así lo usa el usuario);
                     concede acceso si rol='Master' o acceso_<modulo>=1;
                     guarda sesión y redirige al dashboard del módulo.
includes/
  dbconnection.php   PDO. Crea la BD si no existe. GITIGNORED (credenciales).
  dbconnection.php.ejemplo   Plantilla versionada.
dao/
  <modulo>_catalogos.php   Funciones obtenerX($pdo) reutilizables (catálogos).
<modulo>_dashboard.php     Menú con tarjetas: Registro / Historial / Estadísticas.
<modulo>_registro.php      Alta (formulario Kobo + validación + condicionales).
<modulo>_historial.php     Listado con buscador, exportar, y acciones por fila.
<modulo>_editar.php        Edición (precarga desde BD + UPDATE).
<modulo>_eliminar.php      Borrado por id.
<modulo>_pdf.php           Reporte imprimible del listado (window.print).
<modulo>_exportar_excel.php  Descarga .xls (HTML + BOM, sin librería).
<modulo>_descargar_pdf.php   Planilla individual imprimible por registro.
<modulo>_estadisticas.php  Dashboards con Chart.js.
migrations/<modulo>_db.sql Tablas, catálogos, FKs y el nuevo permiso.
```

Convención de nombres: el **module key** (en sesión/permiso) suele ir en
singular (`mcbo_protocolo`, `maracaibo_gaita`) y el **prefijo de archivos**
puede ir en plural (`mcbo_protocolos_*`). Confirma el prefijo con un archivo
existente del módulo antes de crear más; una vez fijado, sé consistente.

## Flujo para crear un módulo nuevo

1. **Obtener la fuente y derivar el modelo.** Pregunta en qué formato está el
   formulario (URL de Kobo/Google Forms, PDF, imágenes, texto, hoja de cálculo)
   y extrae campos, tipos, opciones y reglas condicionales. **Confirma el
   modelo con el usuario antes de crear tablas** (salvo Kobo, las fuentes son
   ambiguas). Ver `references/data-sources.md`.
2. **Diseñar el SQL.** Catálogos + tabla `<modulo>_registros` con FKs + nueva
   columna `acceso_<modulo>` en `usuarios_sistema` + usuario master. Ver
   `references/database.md`.
3. **Registrar el módulo en el index.** Agregar la entrada a `$config_modulos`
   en `index.php`, el `case` de redirección, y el chequeo de permiso +
   redirección en `validacion.php`.
4. **Crear `includes/dbconnection.php`** para la BD nueva (patrón que auto-crea
   la BD). Confirmar que está en `.gitignore`.
5. **Crear `dao/<modulo>_catalogos.php`** con las funciones de catálogo.
6. **Construir las páginas** en orden: dashboard → registro → historial →
   editar → eliminar → exportaciones → estadísticas. Cada una parte de su
   equivalente existente. Ver los references por tipo de página.

## Convenciones no negociables (el "porqué")

- **Base de datos (MySQL 8+/MariaDB, InnoDB, utf8mb4).**
  - **Sin ancho de display en enteros**: usa `int`, `tinyint` (no `int(11)`);
    MySQL 8 emite `1681 deprecated` con el ancho.
  - **Catálogos** = `id` PK AUTO_INCREMENT + `nombre` UNIQUE + `activo`
    tinyint. No agregues `codigo` salvo que necesites una llave estable para
    sincronizar con Kobo; con FKs por `id` el `codigo` sobra.
  - **La tabla de registros usa FKs** a los catálogos (`*_id`), no textos
    sueltos. Índices en las FKs y en `fecha`.
  - Script **idempotente**: `CREATE TABLE IF NOT EXISTS`, `INSERT ... ON
    DUPLICATE KEY UPDATE` para el usuario master.
- **PDO — gotcha crítico**: `includes/dbconnection.php` usa
  `ATTR_EMULATE_PREPARES => false`. Con eso **no puedes reutilizar un
  placeholder nombrado** (`:b`) varias veces; MySQL lanza excepción. En
  búsquedas con LIKE sobre varias columnas usa `:b1..:bN` (bucle) todos con el
  mismo `%valor%`.
- **Seguridad/entrada**: siempre `prepared statements`; `htmlspecialchars` al
  imprimir; textos de negocio en `mb_strtoupper(trim(...), 'UTF-8')` (así lo
  hace el usuario en estos sistemas).
- **Estilo (Bootstrap 5)**: no cambies el sistema visual, adáptalo. Gradiente
  institucional `linear-gradient(90deg, rgba(210,0,90,1) 0%, rgba(22,67,119,1)
  100%)` (clase `.bg-gradient-custom`), acento rosa `#d2005a`, azul `#164377`,
  amarillo `#fdb813`. Formularios: `.seccion-titulo` (borde inferior rosa) y
  `.bloque-seccion` (bloque gris con borde izquierdo rosa). Navbar con
  `imagenes/alcaldia-maracaibo.png` y el lema "Con Mi Gente Se Resuelve♥️".

## Guardas de sesión (en cada página del módulo)

```php
session_start();
require_once 'includes/dbconnection.php';
if (!isset($_SESSION['autentificado']) ||
    ($_SESSION['modulo_activo'] !== '<modulo>' && $_SESSION['rol'] !== 'Master')) {
    header('location:index.php?modulo=<modulo>');
    exit;
}
```

## Patrones por tipo de página (references)

Lee el reference correspondiente antes de construir cada página; traen las
plantillas reales extraídas de los proyectos:

- `references/database.md` — SQL de catálogos, FKs, permiso, e integración con
  `usuarios_sistema`.
- `references/data-sources.md` — cómo derivar el modelo de datos desde
  cualquier fuente (Kobo, PDF, imágenes, Google Forms, texto/Word, hoja de
  cálculo o descripción verbal) y mapearlo al esquema.
- `references/forms.md` — registro y editar: PRG con sesión, `validarFormulario()`
  en cliente, campos condicionales (toggles + reset), placeholders, asteriscos.
- `references/listing-and-export.md` — historial (JOINs + buscador en vivo +
  acciones por fila) y exportaciones PDF (window.print) / Excel (.xls + BOM).
- `references/statistics.md` — estadísticas con Chart.js (helpers `safeCount`/
  `fetchAllAssoc`, `renderChart`, `tooltipConfig`, doughnuts y barras).

## Errores comunes a evitar

- Bloquear el envío con `required` nativo y que no se vea el mensaje: usa
  `novalidate` + `validarFormulario()` (ver forms.md).
- Campos condicionales que quedan visibles al "Limpiar Campos": re-evalúa los
  toggles en el evento `reset` con `setTimeout(...,0)`.
- Reutilizar `:placeholder` en consultas (rompe con prepares reales).
- Fiarte al 100% de la fuente: puede traer errores o vacíos (una regla `relevant`
  imposible en Kobo, un PDF que no marca qué es obligatorio, una foto que corta
  una sección). Implementa la **intención**, confirma supuestos y avísale al
  usuario.
- Enlaces del dashboard a páginas que aún no existen: crea la página o avisa.
