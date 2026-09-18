# Skill: servicio-comunitario

Skill de [Claude Code](https://claude.com/claude-code) para construir y extender
**módulos web PHP** de los sistemas de la **Alcaldía de Maracaibo** desarrollados
como parte de un servicio comunitario.

Captura la arquitectura y las convenciones ya probadas en los proyectos
`mcbo-protocolo` y `maracaibo-gaita` para poder crear un módulo completo
(registro, historial, edición, borrado, exportación y estadísticas) de forma
rápida y consistente, partiendo de **cualquier fuente de datos**: un formulario
de KoboToolbox, un PDF, fotos o escaneos de una planilla, un Google Forms, un
documento de texto/Word, una hoja de cálculo, o incluso una descripción verbal.

## Qué incluye

```
servicio-comunitario/
├── SKILL.md                        # Arquitectura, flujo y convenciones
└── references/
    ├── data-sources.md             # Extraer el modelo desde cualquier fuente
    ├── database.md                 # SQL: catálogos, FKs, permisos, DAO, PDO
    ├── forms.md                    # Registro/editar: PRG, validación, condicionales
    ├── listing-and-export.md       # Historial + exportación PDF/Excel
    └── statistics.md               # Estadísticas con Chart.js
```

También se incluye `servicio-comunitario.skill` (el mismo skill empaquetado en
un solo archivo, listo para instalar).

## Arquitectura que asume

Plataformas **multi-módulo**: un mismo login sirve a varios módulos.

- `index.php` — selector de módulo + login.
- `validacion.php` — verifica el permiso `acceso_<modulo>` en `usuarios_sistema`.
- `includes/dbconnection.php` — conexión PDO (auto-crea la BD; no se versiona).
- `dao/<modulo>_catalogos.php` — funciones reutilizables de catálogos.
- Páginas por módulo con prefijo `<modulo>_*.php` (dashboard, registro,
  historial, editar, eliminar, pdf, exportar_excel, descargar_pdf, estadisticas).
- Estilo Bootstrap 5 institucional (gradiente rosa `#d2005a` → azul `#164377`).

## Instalación

### Opción A — archivo empaquetado
Abre `servicio-comunitario.skill` en la app de Claude y pulsa **Save skill**.

### Opción B — carpeta manual
Copia la carpeta `servicio-comunitario/` a tu directorio de skills:

- **Windows:** `C:\Users\<usuario>\.claude\skills\`
- **macOS/Linux:** `~/.claude/skills/`

## Uso

Se activa solo cuando trabajas en estos proyectos o en cualquiera con la misma
arquitectura, o puedes invocarlo con `/servicio-comunitario`. Pídele cosas como
"crea un módulo nuevo para este formulario", "agrega el historial" o "hazme las
estadísticas".

## Licencia

MIT — ver [LICENSE](LICENSE).
