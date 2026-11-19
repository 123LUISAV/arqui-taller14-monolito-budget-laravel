# Bitácora de trabajo — Taller 14

Registro cronológico de lo que hice (hora local del equipo, 18/11/2026). Las rutas del código analizado están en `C:\Luisa f\arqui\_trabajo\butget_laravel` (no se publica). Los scripts de captura (Playwright) y de generación del documento viven fuera del repositorio, en `_trabajo/t14-tools`.

| Hora | Paso | Comando / acción | Resultado y decisión |
|---|---|---|---|
| 16:04 | Revisión de requisitos | Leí `composer.json`, `package.json`, `README.md`, `.env.example`, `docker-compose.yml` | Requiere PHP ^8.1 y `ext-calendar`, Laravel 10, Livewire 2, Vite 4 + Vue 2; README sugiere `budget:install` |
| 16:04 | Entorno | `php -v`, `composer --version`, `node -v`, `php -m` | PHP 8.2.12, Composer 2.9.5, Node 24.19, extensión `calendar` presente |
| 16:04 | Puerto 3306 | `netstat -ano \| grep 3306` | Libre, MariaDB no estaba corriendo |
| 16:04 | Clonado | `git clone https://github.com/fmoreno4/butget_laravel.git` | Commit `c9d1f3e` |
| 16:04 | Base de datos | `mysqld.exe --defaults-file=... --standalone` (segundo plano) y `SHOW DATABASES` | MariaDB 10.4.32. **Ya existía `budget_laravel` (23 tablas) que no es mía**: no la toqué; decidí crear `budget_taller14` |
| 16:04 | Crear BD | `CREATE DATABASE budget_taller14 CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci` | Correcto |
| 16:04 | Dependencias PHP | `composer install --no-interaction` | Correcto, 117 paquetes. Aviso de PSR-4 en `tests/Unit/Actions/SendVerificationMailTest.php` |
| 16:06 | Configuración | `cp .env.example .env` + `sed` (`DB_DATABASE=budget_taller14`, `DB_USERNAME=root`, `DB_PASSWORD=` vacío, `APP_URL=http://127.0.0.1:8000`) | Correcto |
| 16:06 | Clave | `php artisan key:generate` | Correcto |
| 16:06 | Migraciones | `php artisan migrate` | 52 migraciones `DONE`; 23 tablas; 50 monedas. No usé `db:seed` ni `backup.sql` (datos de otra persona/entorno) |
| 16:06 | Frontend | `npm install`; `npm run build` | 192 paquetes; avisos (Vue 2 EOL, esbuild, chunk 602 kB); build correcto |
| 16:06 | Enlace de storage | `php artisan storage:link` | Correcto |
| 16:06 | Servidor | `php artisan serve --host=127.0.0.1 --port=8000` (segundo plano) | `/` responde 302 a `/login`, `/login` 200 |
| 16:06 | Herramientas de captura | `npm i playwright docx` en `_trabajo/t14-tools` | Correcto; uso Chrome instalado (`channel: 'chrome'`) |
| 16:07 | Primer registro | Playwright: abrir `/`, `/register`, enviar el formulario | **Error E1:** 500 con `530 5.7.1 Authentication required` (SMTP). Mi script además falló al elegir la moneda por una etiqueta incorrecta (E5) |
| 16:07 | Corrección E1 | `echo 'MAIL_MAILER=log' >> .env` | Los correos pasan a `storage/logs/laravel.log` |
| 16:07 | Reintento | Repetí el registro | "The email has already been taken": el 500 había dejado el usuario creado (E1b) |
| 16:08 | Diagnóstico | Consultas SQL sobre `users`, `user_space`, `spaces`, `widgets` | 1 usuario, 1 espacio, 3 widgets huérfanos. Limpié esas filas (la BD es mía) y repetí |
| 16:08 | Registro y login | Registro correcto; logout; login con clave errónea y luego correcta | Funciona; mensaje "Failed to login" en el caso erróneo |
| 16:08 | Etiquetas | Crear Comida, Transporte, Servicios | Primer intento sin efecto: mi selector pulsó el botón *resend* del aviso (E5); corregido por `action` del formulario |
| 16:09 | Transacciones | Asistente de `/transactions/create`: 1 ingreso y 4 gastos | Funciona. Una pasada duplicó el ingreso por reintento de mi script (luego borrado). El campo fecha del asistente no tomó mi valor y quedó la fecha de hoy |
| 16:11 | Editar y borrar | `/spendings/4/edit`; papelera sobre el ingreso duplicado | Editado; borrado lógico (`deleted_at`) |
| 16:11 | Hallazgo E2 | `GET /spendings/create` | 404 por el orden de rutas en `routes/web.php` |
| 16:11 | Presupuestos y deudas | Presupuesto de 400.000 en Comida; deuda de 500.000 y abono de 100.000 | Funciona: 250.000 de 400.000 gastados; saldo 400.000 |
| 16:12 | Reportes, actividad, ajustes, importaciones | Recorrido de pantallas | Cargan; se observan claves sin traducir (`models.debts`, `fields.type`) |
| 16:12 | Livewire | Mover un widget en `/settings/dashboard` | `POST /livewire/message/widget-list`; el orden cambia |
| 16:12 | Pruebas | `CREATE DATABASE budget_taller14_testing`; `phpunit` | **E3:** 52 errores y 6 fallos por tablas inexistentes |
| 16:13 | Corrección E3 | `php artisan migrate --database=testing --force` | Esquema creado |
| 16:13 | Pruebas | `php vendor/bin/phpunit --testdox` | **E4:** 92 pruebas, 162 aserciones, 3 fallos (límite de reenvío comentado, texto en español, tasas de cambio) |
| 16:14 | Verificación de cuenta | Visita a `/verify/{token}` | El aviso desaparece |
| 16:14 | Rutas | `php artisan route:list --except-vendor` | 92 rutas |
| 16:15 | Análisis de código | Lectura de rutas, kernel, middlewares, controladores, repositorios, modelos, eventos, vistas, `AppServiceProvider` | Insumo de las secciones 6 a 8 |
| 16:16 | Diagramas | `npx @mermaid-js/mermaid-cli` con Chrome del sistema | C4 niveles 1 a 3 en `.png` y `.svg` |
| 16:20 | Documento | `md2docx.mjs` (librería `docx`) y Word COM (`SaveAs2`, formato 17) | DOCX y PDF de 23 páginas con tabla de contenido actualizada |
| 16:2x | Cierre | Detener `php artisan serve` y el `mysqld` iniciado por mí | Ver sección final |

## Decisiones principales

1. **Base propia:** `budget_taller14` (y `budget_taller14_testing`) en lugar de reutilizar `budget_laravel`, que ya existía y no era mía.
2. **Sin seeder ni `backup.sql`:** para probar el registro real y no cargar datos de otra persona.
3. **Correo en modo `log`:** solución mínima de desarrollo para el error SMTP; no se editó código del proyecto.
4. **Código ajeno sin modificar:** los hallazgos (404 por orden de rutas, pruebas fallidas, claves sin traducir) se reportan, no se corrigen.
5. **Datos de prueba ficticios:** usuario `luisa.velandia@example.com`; la contraseña de prueba solo existe en la base local de desarrollo, que no se publica.

## Evidencia

Capturas del navegador (`01` a `38`) y de la terminal (`T1` a `T8`) en [`evidencias/`](evidencias/). Las capturas de terminal son renderizados de la salida real de los comandos (se tomó de la bitácora de ejecución y se compuso como imagen).
