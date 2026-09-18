# Base de datos — catálogos, FKs y permisos

Motor objetivo: **MySQL 8+ / MariaDB 10.4+**, InnoDB, `utf8mb4`.
Enteros **sin ancho** (`int`, `tinyint`) para evitar el warning 1681.

## usuarios_sistema (compartida entre módulos)

Cada módulo agrega su propia columna `acceso_<modulo>`. `rol='Master'` puede
todo. El script debe crearla si no existe y agregar el permiso de forma segura.

```sql
CREATE TABLE IF NOT EXISTS `usuarios_sistema` (
  `id` int NOT NULL AUTO_INCREMENT,
  `usuario` varchar(100) NOT NULL,
  `password` varchar(255) NOT NULL,
  `nombre_completo` varchar(200) DEFAULT NULL,
  `rol` varchar(50) DEFAULT 'Usuario',
  `acceso_<modulo>` tinyint DEFAULT 0,
  PRIMARY KEY (`id`),
  UNIQUE KEY `usuario` (`usuario`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

-- Si la tabla ya existe de otro módulo, agregar la columna aparte:
-- ALTER TABLE `usuarios_sistema` ADD COLUMN `acceso_<modulo>` tinyint DEFAULT 0;

INSERT INTO `usuarios_sistema`
  (`usuario`, `password`, `nombre_completo`, `rol`, `acceso_<modulo>`)
VALUES
  ('<modulo>', '<clave>', '<NOMBRE DIRECCIÓN>', 'Master', 1)
ON DUPLICATE KEY UPDATE `acceso_<modulo>` = 1;
```

## Catálogos (uno por cada select_one del Kobo)

Forma canónica: `id` + `nombre` UNIQUE + `activo`. **Sin `codigo`** salvo que
haya que sincronizar con llaves externas de Kobo.

```sql
CREATE TABLE IF NOT EXISTS `<catalogo>` (
  `id` int NOT NULL AUTO_INCREMENT,
  `nombre` varchar(150) NOT NULL,
  `activo` tinyint NOT NULL DEFAULT 1,
  PRIMARY KEY (`id`),
  UNIQUE KEY `nombre` (`nombre`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

INSERT INTO `<catalogo>` (`nombre`) VALUES ('OPCIÓN A'), ('OPCIÓN B');
```

### Catálogo dependiente (relación entre catálogos)

Ej. parroquias que pertenecen a un municipio. Usa FK + una variable para
resolver el padre por nombre (robusto ante el orden de AUTO_INCREMENT):

```sql
CREATE TABLE IF NOT EXISTS `parroquias` (
  `id` int NOT NULL AUTO_INCREMENT,
  `municipio_id` int NOT NULL,
  `nombre` varchar(150) NOT NULL,
  `activo` tinyint NOT NULL DEFAULT 1,
  PRIMARY KEY (`id`),
  UNIQUE KEY `nombre` (`nombre`),
  KEY `idx_parroquia_municipio` (`municipio_id`),
  CONSTRAINT `fk_parroquia_municipio`
    FOREIGN KEY (`municipio_id`) REFERENCES `municipios` (`id`)
    ON UPDATE CASCADE ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

SET @padre_id := (SELECT `id` FROM `municipios` WHERE `nombre` = 'MARACAIBO');
INSERT INTO `parroquias` (`municipio_id`, `nombre`) VALUES (@padre_id, 'PQ. ...');
```

## Tabla principal de registros

Un registro = una carga del formulario. Cada `select_one` del Kobo se guarda
como `*_id` con FK; los campos condicionales van NULL cuando no aplican; los
booleanos Sí/No como `tinyint`.

```sql
CREATE TABLE IF NOT EXISTS `<modulo>_registros` (
  `id` int NOT NULL AUTO_INCREMENT,
  `<catalogo>_id` int NOT NULL,               -- FK
  `<condicional>` varchar(255) DEFAULT NULL,  -- NULL si no aplica
  `<booleano>` tinyint NOT NULL DEFAULT 0,    -- 1=Sí, 0=No
  `fecha` date NOT NULL,
  `comentario` text DEFAULT NULL,
  `fecha_registro` datetime NOT NULL DEFAULT current_timestamp(),
  `usuario_registra` varchar(100) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_fecha` (`fecha`),
  KEY `idx_reg_<catalogo>` (`<catalogo>_id`),
  CONSTRAINT `fk_reg_<catalogo>`
    FOREIGN KEY (`<catalogo>_id`) REFERENCES `<catalogo>` (`id`)
    ON UPDATE CASCADE ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Orden en el .sql**: crea y puebla los catálogos padre antes que los hijos, y
todos los catálogos antes que `<modulo>_registros` (las FKs lo exigen).
Envuélvelo en `START TRANSACTION; ... COMMIT;` y encabézalo con
`SET SQL_MODE="NO_AUTO_VALUE_ON_ZERO"; SET NAMES utf8mb4;`.

## Reglas de negocio que el FK no puede forzar

Ej. "si municipio = Maracaibo, parroquia obligatoria; si no, NULL". Un `CHECK`
no puede hacer subconsultas ni conocer el id de Maracaibo de forma estable, así
que **valida esa regla en la aplicación** (PHP + JS del formulario), no en la
BD. Documenta la intención con un comentario en la columna.

## dbconnection.php (auto-crea la BD; NO versionar)

```php
<?php
$host='localhost'; $db='<nombre_bd>'; $user='root'; $password='<clave>'; $charset='utf8mb4';
$dsn = "mysql:host=$host;dbname=$db;charset=$charset";
$options = [
  PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
  PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
  PDO::ATTR_EMULATE_PREPARES   => false,
];
try {
  $pdo = new PDO($dsn, $user, $password, $options);
} catch (\PDOException $e) {
  if ($e->getCode() === 1049 || strpos($e->getMessage(), 'Unknown database') !== false) {
    $pdoRoot = new PDO("mysql:host=$host;charset=$charset", $user, $password, $options);
    $pdoRoot->exec("CREATE DATABASE IF NOT EXISTS `$db` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;");
    $pdo = new PDO($dsn, $user, $password, $options);
  } else { throw new \PDOException($e->getMessage(), (int)$e->getCode()); }
}
```

Confirma que `.gitignore` incluye `includes/dbconnection.php` (`git check-ignore`).

## DAO de catálogos: dao/<modulo>_catalogos.php

Funciones reutilizables por todas las páginas (registro, editar, historial,
estadísticas, exportaciones). Devuelven arrays `[{id, nombre}]` o `[]` ante
error.

```php
<?php
if (session_status() === PHP_SESSION_NONE) session_start();
require_once __DIR__ . '/../includes/dbconnection.php';

function obtener<Catalogo>($pdo) {
  try {
    return $pdo->query("SELECT id, nombre FROM <catalogo> WHERE activo = 1 ORDER BY nombre ASC")
               ->fetchAll(PDO::FETCH_ASSOC);
  } catch (PDOException $e) { return []; }
}

function obtenerRegistroPorId($pdo, $id) {
  try {
    $stmt = $pdo->prepare("SELECT * FROM <modulo>_registros WHERE id = :id LIMIT 1");
    $stmt->execute([':id' => $id]);
    return $stmt->fetch(PDO::FETCH_ASSOC);
  } catch (PDOException $e) { return null; }
}
```
