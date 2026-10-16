# CI Pipeline para Laravel

[![Laravel CI Pipeline](https://github.com/CrisSFV/CI_Pipeline/actions/workflows/ci.yml/badge.svg)](https://github.com/CrisSFV/CI_Pipeline/actions/workflows/ci.yml)

## Objetivo

Automatizar la instalación de dependencias, la verificación del código y la ejecución de
pruebas de una aplicación Laravel mediante GitHub Actions, de forma que cada cambio enviado
a la rama `main` se valide automáticamente.

## Tecnologías

- PHP 8.2
- Laravel 11
- Composer 2
- PHPUnit 10
- Laravel Pint (verificación de estilo)
- Git y GitHub
- GitHub Actions

## Requisitos

- PHP compatible con el proyecto (`^8.2`, según `composer.json`) con las extensiones
  `mbstring`, `pdo_sqlite`, `sqlite3`, `bcmath` y `fileinfo`
- Composer
- Git
- Acceso al repositorio de GitHub

## Instalación local

1. Clonar el repositorio:
   ```bash
   git clone https://github.com/CrisSFV/CI_Pipeline.git
   cd CI_Pipeline
   ```
2. Instalar dependencias: `composer install`
3. Copiar el archivo de entorno: `cp .env.example .env`
4. Generar la clave de la aplicación: `php artisan key:generate`
5. Configurar la base de datos requerida. Las pruebas usan SQLite en memoria (definido en
   `phpunit.xml`), por lo que no necesitan MySQL.
6. Ejecutar las migraciones si son necesarias: `php artisan migrate`
7. Ejecutar las pruebas: `php artisan test`

> En Windows con XAMPP, `php` y `composer` deben estar en el `PATH` para poder ejecutar
> estos comandos desde cualquier terminal.

## Integración continua

El workflow [`.github/workflows/ci.yml`](.github/workflows/ci.yml) (**Laravel CI Pipeline**)
se ejecuta al enviar cambios (`push`) a la rama `main` o al crear un `pull_request` dirigido
a ella. El job `laravel-tests` corre en `ubuntu-latest` con un límite de 10 minutos y
permisos de solo lectura sobre el repositorio.

| # | Paso | Función |
|---|------|---------|
| 1 | Obtener codigo | `actions/checkout@v4` descarga el código del repositorio. |
| 2 | Configurar PHP | `shivammathur/setup-php@v2` instala PHP 8.2 y las extensiones necesarias. |
| 3 | Validar Composer | `composer validate` verifica `composer.json` y `composer.lock`. |
| 4 | Instalar dependencias | `composer install` instala las versiones fijadas en `composer.lock`. |
| 5 | Preparar entorno Laravel | Copia `.env.example` a `.env`, genera `APP_KEY` y limpia la configuración. |
| 6 | Verificar estilo de codigo (Pint) | `vendor/bin/pint --test` falla si algún archivo PHP no cumple el estilo de Laravel. |
| 7 | Ejecutar pruebas automatizadas | `php artisan test` con `APP_ENV=testing` y SQLite en memoria. |

Si cualquier paso termina con un código distinto de `0`, el job se marca como fallido y
GitHub muestra el commit con una ✗ roja.

No se incluyó la etapa de construcción del frontend (`npm ci` / `npm run build`) porque el
proyecto no tiene `package-lock.json` y no compila recursos con Vite. La guía indica no
agregar ese paso en ese caso.

## Pruebas

| Prueba | Archivo | Qué verifica | Resultado esperado |
|--------|---------|--------------|--------------------|
| `test_that_true_is_true` | `tests/Unit/ExampleTest.php` | Prueba unitaria base de Laravel | PASS |
| `test_the_application_returns_a_successful_response` | `tests/Feature/ExampleTest.php` | `GET /` responde HTTP 200 | PASS |
| `test_home_page_is_accessible` | `tests/Feature/HomePageTest.php` | `GET /` responde con un código de éxito (2xx) | PASS |

`HomePageTest` hace una petición HTTP a la ruta pública `/` (definida en `routes/web.php`,
que devuelve la vista `welcome`) y comprueba con `assertSuccessful()` que la respuesta sea
exitosa. Es la prueba que se modifica para provocar el fallo controlado.

Resultado local:

```
   PASS  Tests\Unit\ExampleTest
  ✓ that true is true
   PASS  Tests\Feature\ExampleTest
  ✓ the application returns a successful response
   PASS  Tests\Feature\HomePageTest
  ✓ home page is accessible

  Tests:    3 passed (3 assertions)
```

<!-- Agregar aquí la captura de las pruebas locales: ![Pruebas locales](docs/img/pruebas-locales.png) -->

## Evidencias

| Ejecución | Commit | Resultado | Enlace |
|-----------|--------|-----------|--------|
| A. Ejecución exitosa | _pendiente_ | ✅ Success | _pendiente_ |
| B. Ejecución fallida controlada | _pendiente_ | ❌ Failure | _pendiente_ |
| C. Ejecución exitosa tras la corrección | _pendiente_ | ✅ Success | _pendiente_ |

Historial completo: <https://github.com/CrisSFV/CI_Pipeline/actions>

### A. Ejecución exitosa

Con las pruebas correctas, los 7 pasos del job `laravel-tests` terminan en verde y el paso
**Ejecutar pruebas automatizadas** muestra `Tests: 3 passed`.

### B. Ejecución fallida controlada

En `tests/Feature/HomePageTest.php` se cambió temporalmente la aserción:

```php
$response->assertStatus(404);
```

La ruta `/` responde correctamente con HTTP 200, así que la aserción no se cumple. El paso
**Ejecutar pruebas automatizadas** termina con código `1` y el log muestra:

```
FAILED  Tests\Feature\HomePageTest > home page is accessible
Expected response status code [404] but received 200.
Tests:    1 failed, 2 passed
```

**Interpretación:** el error no está en la aplicación ni en el pipeline, sino en la
expectativa de la prueba. El pipeline detectó la diferencia y bloqueó el commit con una ✗
roja.
**Solución:** restaurar la aserción correcta (`assertSuccessful()`), comprobarla localmente
con `php artisan test --filter=HomePageTest` y volver a hacer `push`.

### C. Ejecución exitosa tras la corrección

Con la aserción restaurada, el workflow se ejecuta de nuevo automáticamente y vuelve a
terminar en verde (`3 passed`). Esto confirma el ciclo completo: detección, corrección y
validación.

## Conclusiones

- El pipeline valida automáticamente cada cambio en `main`: dependencias reproducibles
  (gracias a `composer.lock`), estilo de código y pruebas, sin intervención manual.
- La ejecución fallida demostró que el pipeline no solo se ejecuta, sino que **detecta
  errores** y los comunica con un log que señala el archivo, la prueba y la causa exacta
  (`Expected response status code [404] but received 200`).
- Usar SQLite en memoria en `phpunit.xml` y en el workflow elimina la dependencia del MySQL
  de XAMPP, así que las pruebas dan el mismo resultado en Windows y en Ubuntu.
- La versión de PHP del workflow (8.2) coincide con la exigida por `composer.json`
  (`^8.2`) y con la del entorno local, lo que evita errores de `requirements could not be
  resolved`.
- Beneficios de la integración continua: retroalimentación rápida en cada `push`,
  detección temprana de regresiones, un entorno de validación limpio e idéntico en cada
  ejecución, y un historial verificable de qué commits pasan o fallan.
