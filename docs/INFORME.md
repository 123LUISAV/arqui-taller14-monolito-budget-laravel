# Informe técnico de instalación y análisis de aplicación monolítica

**Aplicación analizada:** Budget (`fmoreno4/butget_laravel`) — Laravel 10, Livewire 2, Tailwind/Vite
**Asignatura:** Arquitectura de Software — Taller 14
**Estudiante:** Luisa Fernanda Velandia Salamanca (código U00179371)
**Institución:** Universidad Autónoma de Bucaramanga (UNAB)
**Docentes:** MSc. Fabian Suarez, MSc. Javier Pinzon, MSc. Feisar Moreno
**Fecha:** noviembre de 2026

---

## 1. Introducción

Este informe documenta el trabajo que realicé sobre **Budget**, una aplicación web de control de finanzas personales (ingresos, gastos, etiquetas, presupuestos, deudas y reportes) escrita en PHP con el framework Laravel 10. El propósito del taller es doble: demostrar que puedo reproducir de principio a fin la instalación de un sistema ajeno a partir de su repositorio y usar ese sistema en funcionamiento como caso real para analizar cómo está construido y qué decisiones de arquitectura reflejan sus carpetas, clases y rutas.

El proceso que seguí fue: (1) verificar qué herramientas tenía instaladas en mi equipo Windows 11 y cuáles exigía el proyecto; (2) clonar el repositorio e instalar las dependencias de PHP (Composer) y de frontend (npm); (3) configurar el archivo `.env` y crear una base de datos propia; (4) ejecutar las migraciones, compilar los assets y levantar el servidor local; (5) recorrer la aplicación en un navegador real (registro, inicio de sesión, CRUD de transacciones, presupuestos, deudas, reportes y ajustes), capturando evidencia; y (6) leer el código fuente para describir la arquitectura y modelarla con C4 en sus tres primeros niveles (contexto, contenedores y componentes).

**Objetivo.** Instalar y ejecutar localmente Budget y determinar, con base en el código y no en teoría, si es una aplicación monolítica, cómo se separa en capas, dónde reside la lógica de negocio y qué problemas y mejoras se identifican frente a la arquitectura en capas (N-tier) vista en clase.

**Alcance.** Instalación en un entorno local de desarrollo (sin Docker), ejecución de las funcionalidades principales de la interfaz web, análisis estático del código en el commit `c9d1f3e` y ejecución de su suite de pruebas. No incluye despliegue en producción, pruebas de carga ni un análisis de seguridad exhaustivo.

**Importancia de comprender la arquitectura.** Instalar un sistema es solo un primer paso; entender cómo está organizado determina qué tan fácil será mantenerlo, probarlo, escalarlo o migrarlo. Un monolito bien estructurado puede ser una excelente decisión, pero uno con las capas mezcladas acumula deuda técnica. Los errores que encontré durante la ejecución (sección 4) y los hallazgos de la sección 8 muestran cómo una decisión arquitectónica se manifiesta en problemas concretos.

## 2. Preparación del entorno

Antes de instalar nada revisé el `composer.json`, el `package.json`, el `README.md`, el `.env.example` y el `docker-compose.yml` del repositorio para conocer los requisitos que declara, y luego comprobé con comandos cuáles ya cumplía mi equipo. Esa verificación evitó instalar software innecesario.

| Componente | Versión encontrada | Por qué fue necesario | Cómo lo verifiqué |
|---|---|---|---|
| Sistema operativo | Windows 11 Pro | Plataforma de trabajo; usé Git Bash (MINGW64) | Información del sistema |
| Servidor local | XAMPP (aporta PHP y MariaDB). Para ejecutar la app usé el servidor embebido de Laravel, `php artisan serve` | La app necesita PHP y una base de datos; no requiere Apache en desarrollo | Existencia de `C:\xampp\php` y `C:\xampp\mysql\bin\mysqld.exe` |
| PHP | 8.2.12 (ZTS, x64) | `composer.json` exige `php: ^8.1` | `php -v` |
| Extensiones PHP | `calendar`, `pdo_mysql`, `mbstring`, `openssl`, `intl`, `zip`, `fileinfo`, `curl`, `tokenizer` | `composer.json` exige `ext-calendar` (el dashboard usa `cal_days_in_month`); `pdo_mysql` para la BD; las demás las requiere Laravel | `php -m` |
| Composer | 2.9.5 | Instala las dependencias PHP fijadas en `composer.lock` | `composer --version` |
| Node.js / npm | v24.19.0 / 11.17.0 | Vite compila Vue 2, Tailwind y Sass hacia `public/build` | `node -v`, `npm -v` |
| Motor de BD | MariaDB 10.4.32 (XAMPP), compatible con MySQL | El `.env.example` y el `backup.sql` indican MySQL (el volcado viene de MySQL 8.0.37) | `mysqld.exe --version`, `SELECT VERSION()` |
| Navegador | Google Chrome | Verificación visual y capturas con Playwright | Instalado |

![Verificación del entorno](evidencias/T1-terminal-entorno.png)

Dos comprobaciones previas fueron relevantes. Primero, que **`ext-calendar` estuviera activa**: de no estarlo, `composer install` habría fallado por requisito de plataforma. Segundo, **revisar si MariaDB ya corría** (`netstat -ano | grep 3306`): el puerto estaba libre, por lo que lo inicié yo con `mysqld.exe --defaults-file=... --standalone` en segundo plano y lo detuve al terminar. Al listar las bases existentes vi que **ya existía una llamada `budget_laravel`** con 23 tablas que no era mía; en lugar de reutilizarla o borrarla, creé una base nueva y propia, `budget_taller14`, para no tocar datos ajenos.

El proyecto incluye Docker (`docker-compose.yml`, `docker/Dockerfile`), pero no tengo Docker instalado; XAMPP era suficiente y es el camino más cercano a un entorno de desarrollo típico en Windows.

## 3. Proceso de instalación

Todos los comandos se ejecutaron desde Git Bash y quedaron registrados en la [bitácora](BITACORA.md).

| # | Comando | Propósito | Resultado |
|---|---|---|---|
| 1 | `git clone https://github.com/fmoreno4/butget_laravel.git` | Obtener el código (commit `c9d1f3e`) | Correcto |
| 2 | `mysqld.exe --defaults-file=/c/xampp/mysql/bin/my.ini --standalone` (segundo plano) | Iniciar el motor de base de datos | Escuchando en 3306 |
| 3 | `mysql.exe -uroot -e "CREATE DATABASE budget_taller14 CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci"` | Crear la base propia con codificación completa | Correcto |
| 4 | `composer install --no-interaction` | Instalar 117 paquetes PHP según `composer.lock`; al final ejecuta `package:discover` y `app:update-version-file` | Correcto, sin errores |
| 5 | `cp .env.example .env` y edición con `sed` | Configuración local: `DB_DATABASE=budget_taller14`, `DB_USERNAME=root`, `DB_PASSWORD=` (vacía, valor por defecto de XAMPP), `APP_URL=http://127.0.0.1:8000` | Correcto |
| 6 | `php artisan key:generate` | Generar `APP_KEY`, necesaria para cifrar sesiones y cookies | Correcto |
| 7 | `php artisan migrate` | Crear las tablas y cargar 50 monedas (varias migraciones insertan datos en `currencies`) | 52 migraciones, todas `DONE` (23 tablas) |
| 8 | `npm install` | Instalar dependencias de frontend (192 paquetes) | Correcto, con avisos |
| 9 | `npm run build` | Compilar Vue/Tailwind/Sass con Vite hacia `public/build` | Correcto (1395 módulos) |
| 10 | `php artisan storage:link` | Enlazar `public/storage` con `storage/app/public` (avatares y adjuntos) | Correcto |
| 11 | `php artisan serve --host=127.0.0.1 --port=8000` | Levantar el servidor de desarrollo | `GET /` responde 302 hacia `/login`; `/login` responde 200 |

![composer install](evidencias/T2-terminal-composer-install.png)
![.env y key:generate](evidencias/T3-terminal-env-key.png)
![migraciones](evidencias/T4-terminal-migraciones.png)
![npm run build](evidencias/T5-terminal-npm-build.png)
![servidor](evidencias/T6-terminal-serve.png)

**Decisiones.** (a) No ejecuté `php artisan db:seed`: el `DatabaseSeeder` crea un usuario de ejemplo con datos de otra persona, y preferí registrar mi propio usuario por la interfaz para probar el flujo real. (b) No importé `backup.sql`: es un volcado de otro entorno (`budget_laravel_db`) y las migraciones ya construyen el esquema completo. (c) El README del proyecto menciona `php artisan budget:install`, que internamente corre `npm install`, `npm run build`, `cp .env.example .env`, `key:generate` y `storage:link`; hice esos pasos manualmente para poder explicar y verificar cada uno. (d) Las pruebas de la sección 5.3 usan una segunda base, `budget_taller14_testing`.

**Avisos (no son errores).** `composer install` avisó que la clase `Tests\Unit\SendVerificationMailTest` no cumple PSR-4 (el archivo está en `tests/Unit/Actions/` pero el namespace no coincide). `npm install` avisó que Vue 2 llegó a fin de vida (EOL) y que `esbuild` tiene un script de instalación sin aprobar; `npm run build` avisó de un chunk de 602 kB y de `caniuse-lite` desactualizado. Ninguno impidió la instalación.

## 4. Registro de errores y soluciones

Documento solo lo que realmente ocurrió. **Los pasos de instalación (composer, migraciones, npm, build, servidor) no produjeron errores.** Los problemas aparecieron al ejecutar la aplicación y sus pruebas.

**Uso de IA.** Realicé este trabajo apoyándome en un asistente de IA (Claude Code, de Anthropic), que ejecutó comandos, leyó los logs y propuso las correcciones. En cada error indico la consulta que formulé al asistente (resumida, tal como la planteé durante la sesión) y la respuesta clave. Las causas y soluciones las verifiqué reproduciendo el error y comprobando que desaparecía.

| ID | Momento | Error | Causa | Solución |
|---|---|---|---|---|
| E1 | Primer registro de usuario (`POST /register`) | HTTP 500: `UnexpectedResponseException: Expected response code "250" but got code "530", with message "530 5.7.1 Authentication required"` | `RegisterController::store` llama a `SendVerificationMailAction`, que envía un correo por SMTP. El `.env.example` apunta a `smtp.mailtrap.io:2525` con credenciales `null`, y el servidor rechazó la autenticación | Añadir `MAIL_MAILER=log` al `.env` para que Laravel escriba los correos en `storage/logs/laravel.log` |
| E1b | Consecuencia de E1 | Reintentar el registro devolvía "The email has already been taken" | El controlador crea usuario y espacio **antes** de enviar el correo y sin transacción: el 500 dejó un usuario, un espacio y 3 widgets huérfanos | Como la BD es mía, borré esas filas y repetí el registro. Se analiza en la sección 8 |
| E2 | Prueba de URLs del CRUD | `GET /spendings/create` responde 404 | En `routes/web.php` la ruta `/spendings/{spending}` está declarada **antes** que `/spendings/create`; Laravel interpreta `create` como un id y falla el *route model binding*. Lo mismo ocurre con `/earnings/create` | No lo corregí en el código ajeno. La interfaz no se ve afectada porque el menú "Create Transaction" usa `/transactions/create`, un asistente Vue que hace `POST` a `/spendings` y `/earnings`. Queda como hallazgo |
| E3 | Primera ejecución de PHPUnit | 52 errores y 6 fallos: `Table 'budget_taller14_testing.spaces' doesn't exist` | `phpunit.xml` fija `DB_CONNECTION=testing` y esa conexión de `config/database.php` usa `DB_DATABASE` + `_testing`. Esa base (que creé vacía) no tenía tablas | `php artisan migrate --database=testing --force` |
| E4 | Segunda ejecución de PHPUnit | 92 pruebas, 3 fallos | (a) `SendVerificationMailTest::testVerificationRateLimit`: el bloque de límite de reenvío está **comentado** en `SendVerificationMailAction`; (b) `Mail\ResetPasswordTest` espera el texto "Click here to change your password" pero la vista está en español ("Click aquí para cambiar la contraseña."); (c) `FetchConversionRateTest`: la aserción sobre `conversion_rates` esperaba 0 filas y encontró 1 (no profundicé la causa: la prueba usa un servicio externo y una base sin limpieza automática) | No son problemas de instalación sino de la suite del proyecto; los reporto sin modificarlos |
| E5 | Scripts de captura (herramienta mía, no de la app) | Selectores de Playwright que no encontraban el elemento (`Colombian Peso` en vez de `Colombian Peso (COP)`; `form button.button` coincidía con el botón *resend* del aviso superior) | Selectores demasiado genéricos | Usar la etiqueta exacta y selectores por `action` del formulario. Una de esas pasadas creó un ingreso duplicado que luego eliminé desde la interfaz (sirvió para probar el borrado) |

![Excepción SMTP en el navegador](evidencias/04-despues-registro.png)
![Log y estado de la base tras el error](evidencias/T7-terminal-error-smtp.png)
![Reintento: correo ya registrado](evidencias/05-registro-email-duplicado.png)
![404 en /spendings/create](evidencias/19-error-404-spendings-create.png)

### 4.1 Transparencia: consultas a la IA

**Consulta 1 (E1).** *"Me sale un error 500 al registrarme en Budget: 'Expected response code 250 but got 530 Authentication required'. ¿Qué significa y cómo lo soluciono en local sin configurar un correo real?"*
**Respuesta clave:** el código 530 es de SMTP e indica que el servidor exige autenticación; el flujo de registro envía el correo de verificación y el `.env` apunta a Mailtrap sin credenciales. Para desarrollo basta cambiar el *mailer* a `log`. **Cómo ayudó:** localizó en el log y en `SendVerificationMailAction` el origen del envío y sugirió la solución mínima. Yo comprobé el resultado: tras el cambio el registro redirigió al dashboard.

**Consulta 2 (E1b).** *"Después del error el correo aparece como ya registrado. ¿Qué pasó en la base de datos?"*
**Respuesta clave:** el controlador crea usuario y espacio antes de enviar el correo, sin transacción; el fallo dejó datos parciales. **Cómo ayudó:** me indicó consultar `users`, `user_space` y `spaces` para confirmarlo (1 usuario, 1 espacio, 3 widgets).

**Consulta 3 (E2).** *"Al abrir /spendings/create me da 404 aunque la ruta existe en web.php. ¿Por qué?"*
**Respuesta clave:** el orden de declaración de rutas; `/spendings/{spending}` captura `create` primero. **Cómo ayudó:** me llevó a verificar el orden en `routes/web.php` y a ver que el flujo real usa el asistente de `/transactions/create`.

**Consulta 4 (E3/E4).** *"¿Cómo ejecuto las pruebas del proyecto sin dañar mi base de datos y por qué fallan casi todas?"*
**Respuesta clave:** `phpunit.xml` usa la conexión `testing`, que apunta a otra base (`<DB_DATABASE>_testing`) que hay que crear y migrar. **Cómo ayudó:** evitó que las pruebas tocaran la base de la aplicación y resolvió los 52 errores; los 3 fallos restantes los revisé en el código de cada prueba.

## 5. Ejecución del sistema

**URL:** `http://127.0.0.1:8000` (servidor embebido de Laravel, puerto 8000). La interfaz está parcialmente en español porque `SetLocale` toma el idioma del usuario (`$user->language`). Datos de prueba: usuario ficticio `luisa.velandia@example.com`, moneda COP.

### 5.1 Funcionalidades probadas

| Funcionalidad | Cómo la probé | Resultado | Evidencia |
|---|---|---|---|
| Acceso sin sesión | `GET /` | Redirige a `/login` | ![](evidencias/01-login.png) |
| Registro | Formulario con nombre, correo, contraseña y moneda | Correcto tras corregir E1; crea usuario, espacio ("Luisa Velandia's Space") y 3 widgets | ![](evidencias/03-registro-lleno.png) ![](evidencias/06-dashboard-inicial.png) |
| Inicio de sesión | Credenciales erróneas y luego correctas | Mensaje "Failed to login" y luego acceso al dashboard | ![](evidencias/07-login-credenciales-invalidas.png) |
| Verificación de cuenta | Visité `/verify/{token}` con el token de la BD | Desaparece el aviso de cuenta sin verificar | ![](evidencias/38-cuenta-verificada.png) |
| Etiquetas | Creé Comida, Transporte y Servicios | Correcto | ![](evidencias/09-etiquetas-lista.png) |
| Transacciones (crear) | Asistente Vue de `/transactions/create`: 1 ingreso (3.500.000) y 4 gastos | Correcto; montos guardados en centavos (`350000000`) | ![](evidencias/12-transaccion-ingreso-lleno.png) ![](evidencias/13-transaccion-gasto-lleno.png) |
| Transacciones (listar y filtrar) | `/transactions` | Agrupadas por mes, con filtro por etiqueta | ![](evidencias/14-transacciones-lista.png) |
| Transacciones (detalle y edición) | `/spendings/4` y `/spendings/4/edit` | Descripción y monto actualizados | ![](evidencias/16-gasto-editar-antes.png) ![](evidencias/17-gasto-editado.png) |
| Transacciones (eliminar) | Papelera sobre un ingreso duplicado | Borrado lógico: la fila queda con `deleted_at` | ![](evidencias/18-transacciones-tras-eliminar.png) |
| Presupuestos | Presupuesto mensual de 400.000 para Comida | Muestra 250.000 gastados de 400.000 | ![](evidencias/21-presupuestos-lista.png) |
| Deudas | Deuda "Me prestan" de 500.000 y un abono de 100.000 | Total, abono y saldo calculados (saldo 400.000) | ![](evidencias/24-deudas-lista.png) ![](evidencias/27-deuda-con-abono.png) |
| Dashboard | `/dashboard` | Balance 3.115.000, gastado 385.000, etiquetas más costosas y gráfico diario | ![](evidencias/28-dashboard-con-datos.png) |
| Reportes | Balance semanal y etiquetas más costosas | Correcto | ![](evidencias/29-reportes.png) ![](evidencias/30-reporte-etiquetas.png) |
| Actividad | `/activities` | Registra cada creación (generada por eventos) | ![](evidencias/32-actividad.png) |
| Widgets (Livewire) | En `/settings/dashboard` moví un widget hacia abajo | El orden cambia sin recargar; la petición va a `POST /livewire/message/widget-list` | ![](evidencias/37-livewire-widget-movido.png) |
| Otras pantallas | Ajustes, espacios, importaciones | Cargan correctamente | ![](evidencias/34-ajustes-espacios.png) ![](evidencias/35-importaciones.png) |

Detalles observados: el menú muestra la clave sin traducir `models.debts` y la pantalla de deudas muestra `fields.type`; la consola del navegador advierte que los scripts de Livewire se cargan dos veces. La importación de CSV y las invitaciones a espacios no las ejecuté en profundidad.

### 5.2 Verificación en base de datos

Tras las pruebas comprobé por consulta SQL que los datos coinciden con la interfaz (por ejemplo, `spendings.amount = 7000000` equivale a COP 70.000,00 en pantalla) y que el borrado de transacciones es lógico.

### 5.3 Suite de pruebas

Ejecuté `php vendor/bin/phpunit` (PHPUnit 9.6.20): **92 pruebas, 162 aserciones, 3 fallos** (ver E4).

![PHPUnit](evidencias/T8-terminal-phpunit.png)

## 6. Análisis arquitectónico

### 6.1 ¿Es una aplicación monolítica?

Sí. Un único proyecto Laravel contiene la interfaz, la lógica y el acceso a datos, y se despliega como una sola unidad:

- **Un solo repositorio y un solo proceso:** `php artisan serve` (o el contenedor nginx + php-fpm de `docker/`) sirve todo. No hay servicios independientes ni comunicación entre ellos.
- **Una sola base de datos relacional** compartida por todos los módulos (23 tablas), a la que accede el ORM Eloquent.
- **Vistas renderizadas en el servidor:** 70 plantillas Blade en `resources/views`, con pequeños componentes Vue 2 y Livewire incrustados (`<transaction-wizard>`, `<DatePicker>`, componentes `WidgetList`/`WidgetWizard`).
- **Tareas en segundo plano dentro del mismo código:** `app/Console/Kernel.php` programa `ProcessRecurrings` (diario), `FetchConversionRates` (diario) y `SendWeeklyReports` (viernes 21:00), y la cola usa la tabla `jobs` de la misma base.
- **Dos accesos al mismo núcleo:** rutas web (`routes/web.php`, sesión + CSRF) y rutas API (`routes/api.php`, `ResolveApiKey`), que llaman a los mismos repositorios y modelos. Existe además un prototipo SPA (`/prototype`, `resources/assets/js/prototype`) que consume esa API y está protegido por `CheckIfSpaPrototypeIsEnabled`.

Cifras del repositorio (líneas): controladores 1.995, modelos 963, repositorios 922, acciones 237, políticas 192, vistas Blade 2.597, JavaScript/Vue 1.607, pruebas 2.399.

### 6.2 ¿Hay separación por capas?

Hay una **separación por capas parcial, basada en carpetas**, no en dependencias estrictas:

| Capa (modelo N-tier) | Dónde está | Ejemplos reales |
|---|---|---|
| Presentación | `routes/`, `app/Http/Controllers`, `app/Http/Livewire`, `resources/views`, `resources/assets/js` | `DashboardController`, `WidgetList`, `transactions/create.blade.php`, `TransactionWizard.vue` |
| Lógica de aplicación / negocio | `app/Actions`, `app/Repositories`, `app/Policies`, `app/Http/Requests`, `app/Events`, `app/Jobs` | `CreateUserAction`, `SpendingRepository`, `SpendingPolicy`, `DebtRequest`, `ProcessRecurrings` |
| Datos | `app/Models` (Eloquent), `database/migrations` | `Spending`, `Space`, `Tag`, `Debt` |

La convención es *controlador, repositorio, modelo*. Por ejemplo, `TagController` recibe un `TagRepository` en el constructor y le delega la validación y la creación:

```php
// app/Http/Controllers/TagController.php
$request->validate($this->tagRepository->getValidationRules());
$this->tagRepository->create(session('space_id'), $request->input('name'), $request->input('color'));
```

Pero la separación **no se respeta de manera consistente** (ver 6.3 y sección 8): los controladores también consultan modelos directamente (`Tag::query()->where('space_id', session('space_id'))` en `TagController::index`; `Debt::query()` en `DebtController::index`) y los repositorios dependen de la sesión HTTP.

### 6.3 ¿Dónde está la lógica de negocio?

No está concentrada en una capa; está repartida:

1. **En repositorios** (`app/Repositories`): reglas de validación (`SpendingRepository::getValidationRules()` define `'amount' => 'required|regex:/^\d*(\.\d{2})?$/'`), consultas agregadas (`DashboardRepository::getTotalAmountSpent`, `getDailyBalance`) y estado de presupuestos (`BudgetRepository`).
2. **En modelos** (`app/Models`): cálculos de dominio como `Space::generalBalance()` (ingresos menos gastos), `Space::monthlyBalance()` y `Budget::spent`, y el formateo de montos (`Spending::formattedAmount` divide entre 100).
3. **En controladores:** `SpendingController::store` convierte monedas con `ConversionRateRepository::convert(...)` y transforma el monto con `Helper::rawNumberToInteger`; `ImportController` lee el CSV con `fgetcsv` y `Helper::detectDelimiter`; `RegisterController::store` orquesta la creación de usuario, espacio, rol `admin`, correo y sesión.
4. **En eventos:** los modelos declaran `protected $dispatchesEvents = ['created' => TransactionCreated::class, ...]` y el constructor del evento escribe un registro `Activity` (la auditoría ocurre allí, no en un listener).
5. **En middlewares y proveedores:** `SetLocale` aplica el idioma del usuario y `AppServiceProvider::boot` registra un *view composer* sobre `'*'` que consulta la base en cada vista.

El concepto de dominio central es el **Space** (espacio): casi todas las entidades llevan `space_id` y el espacio activo se guarda en la sesión (`StoreSpaceInSessionAction` ejecuta `session(['space_id' => $spaceId])`). Es la forma de aislar datos entre usuarios y se aplica manualmente en cada consulta y política (`$user->spaces->contains($spending->space_id)`).

### 6.4 Control de acceso

`routes/web.php` agrupa las rutas privadas bajo `middleware => ['auth']` y las de acceso público bajo `guest`; la API usa `resolve-api-key`. La autorización de recursos se hace con políticas (`SpendingPolicy`, `TagPolicy`, `DebtPolicy`...) invocadas con `$this->authorize(...)` (28 usos en controladores).

## 7. Modelo C4

Los diagramas se construyeron a partir de lo observado en el código y en la ejecución. Están en `docs/diagramas/` (fuente Mermaid `.mmd` y exportados a `.png` y `.svg`).

### 7.1 Nivel 1 — Contexto

![C4 nivel 1](diagramas/c4-nivel1-contexto.png)

- **Usuario:** persona que se registra, inicia sesión y gestiona sus finanzas desde el navegador.
- **Cliente de la API:** quien consume `/api/*` con la cabecera `api-key` (modelo `ApiKey`, middleware `ResolveApiKey`). La usa el prototipo SPA y podría usarla una app móvil.
- **Sistema Budget:** el monolito Laravel.
- **Sistemas externos (existen en el código):** un **servidor SMTP** (`app/Mail`: `VerifyRegistration`, `ResetPassword`, `WeeklyReport`...); la **currency-api** en `cdn.jsdelivr.net/gh/fawazahmed0/currency-api@1/...`, consultada por `app/Jobs/FetchConversionRate.php` con Guzzle; y los **CDN de recursos** que el navegador descarga en `layout.blade.php` (Font Awesome Pro, Google Fonts, Chartist). Este último es una dependencia del navegador, no del servidor.

### 7.2 Nivel 2 — Contenedores

![C4 nivel 2](diagramas/c4-nivel2-contenedores.png)

| Contenedor | Tecnología | Responsabilidad |
|---|---|---|
| Navegador web | HTML de Blade, Vue 2.7, Livewire.js, Chartist, Axios (compilados con Vite) | Muestra la interfaz; el asistente de transacciones hace `axios.post('/spendings' ...)`; Livewire envía `POST /livewire/message/...` |
| Aplicación monolítica Laravel | PHP 8.2, Laravel 10.48, Livewire 2, Eloquent | Rutas, middlewares, controladores, repositorios, acciones, modelos, vistas |
| Planificador y cola | `schedule:run` y `queue:work` (cron en `docker/cron.conf`) | Recurrentes, tasas de cambio e informes semanales |
| Base de datos | MySQL 8 (dump) / MariaDB 10.4 (usada), relacional | 23 tablas: `users`, `spaces`, `user_space`, `spendings`, `earnings`, `tags`, `budgets`, `debts`, `debt_details`, `widgets`, `activities`, `jobs`... |
| Almacenamiento de archivos | `storage/app` (sistema de archivos local) | CSV importados (`store('imports')`), avatares (`Storage::put('public/avatars/...')`), adjuntos |

Estructura interna de la aplicación: `app/` (Actions, Console, Enums, Events, Exceptions, Http, Jobs, Mail, Models, Policies, Providers, Repositories, Widgets), `routes/` (web, api, console, channels), `resources/` (views, assets), `database/` (migraciones, seeders, factories), `config/` y `tests/` (Unit, Feature, Browser/Dusk).

### 7.3 Nivel 3 — Componentes

![C4 nivel 3](diagramas/c4-nivel3-componentes.png)

| Componente | Rol | Ejemplos reales |
|---|---|---|
| Rutas | Mapean URL a controlador; 92 rutas | `routes/web.php`, `routes/api.php` |
| Middlewares | Filtran la petición: sesión, CSRF, autenticación, idioma, API key | `EncryptCookies`, `VerifyCsrfToken`, `auth`, `guest` (`RedirectIfAuthenticated`), `SetLocale`, `RefreshSpaceRelations`, `ResolveApiKey` |
| Controladores web | Reciben la petición, validan y devuelven una vista o redirección | `DashboardController`, `TransactionController`, `SpendingController`, `TagController`, `BudgetController`, `DebtController`, `ReportController`, `ImportController`, `SettingsController`, `SpaceController` |
| Controladores API | Versión JSON para clientes con API key | `Api/TransactionController`, `Api/DashboardController`... |
| Componentes Livewire | Interacción sin recarga en ajustes | `WidgetList` (`up`, `down`, `delete`), `WidgetWizard` |
| Vistas | Plantillas Blade y componentes Vue | `layout.blade.php`, `dashboard.blade.php`, `TransactionWizard.vue` |
| Form Requests y Policies | Validación y autorización | `DebtRequest`, `DebtDetailRequest`, `SpendingPolicy`, `SpacePolicy` |
| Actions | Casos de uso pequeños | `CreateUserAction`, `CreateSpaceAction`, `AcceptSpaceInviteAction` |
| Repositorios | Consultas y reglas por entidad (15) | `SpendingRepository`, `DashboardRepository`; `DebtRepository` es una **interfaz** enlazada a `ORMDebtRepository` en `AppServiceProvider::register` |
| Eventos | Auditoría de creación y borrado | `TransactionCreated`, `TagCreated`, `ImportDeleted`, que generan `Activity` |
| Jobs y Mail | Procesos diferidos y correos | `ProcessRecurrings`, `FetchConversionRates`, `SendWeeklyReports`, `VerifyRegistration` |
| Modelos Eloquent | Entidades, relaciones y cálculos | `User`, `Space`, `Spending`, `Earning`, `Tag`, `Budget`, `Debt`, `Widget`, `Activity` |

**Interacción típica (crear un gasto):** el navegador envía `POST /spendings`; pasa por el grupo `web` (sesión, CSRF, `SetLocale`) y `auth`; `SpendingController::store` valida con `SpendingRepository::getValidationRules()`, convierte el monto y llama a `SpendingRepository::create`; `Spending::create` inserta la fila y el modelo dispara `TransactionCreated`, que inserta un registro en `activities`; el controlador redirige al dashboard, que vuelve a consultar los agregados.

## 8. Análisis crítico

### 8.1 ¿Está bien estructurada?

Parcialmente. La organización por carpetas es reconocible y sigue las convenciones de Laravel; se usan políticas para autorización, *Form Requests* en el módulo de deudas, acciones para casos de uso y una suite de 92 pruebas (con 3 fallos). Eso la hace relativamente fácil de recorrer. Sin embargo, frente al modelo de tres capas visto en clase las fronteras son difusas y varias dependencias van en sentido contrario al esperado.

### 8.2 Problemas encontrados

| # | Problema | Evidencia en el código | Efecto |
|---|---|---|---|
| 1 | La capa de negocio/datos depende de la presentación (la sesión HTTP) | `DashboardRepository` usa `session('space_id')`; `CurrencyRepository::getIfConversionRatePresent` también | Los repositorios no se pueden reutilizar en consola o jobs sin simular una sesión |
| 2 | Lógica de negocio en controladores | Conversión de moneda en `SpendingController::store`; lectura de CSV en `ImportController`; orquestación en `RegisterController` | Duplicación (existe también `Api/TransactionController`) y difícil de probar |
| 3 | Sin transacciones y acoplamiento con un servicio externo | En `RegisterController::store` el correo se envía de forma síncrona después de insertar usuario y espacio (E1/E1b) | Un fallo SMTP deja datos huérfanos y devuelve 500 |
| 4 | Eventos con dependencias de HTTP | El constructor de `TransactionCreated` lee `Auth::check()` y `request()->get('apiKey')` y crea la `Activity`; `EventServiceProvider` no registra listeners propios | Lógica de auditoría dentro del evento y no en un *listener* |
| 5 | Consultas ineficientes | `DashboardRepository::getDailyBalance` ejecuta 2 consultas por día del mes (hasta 62); el *view composer* `'*'` de `AppServiceProvider` hace `Space::find` en cada vista renderizada | Costo creciente con el uso |
| 6 | Código muerto | `RefreshSpaceRelations` solo actúa si `session('space')` existe, pero el proyecto solo guarda `space_id` (no encontré ningún lugar que escriba `space`) | Middleware sin efecto |
| 7 | Abstracción aplicada a medias | Solo deudas usa interfaz + `bind` (`DebtRepository` a `ORMDebtRepository`), cuyo método `getDebtsDetailsById` devuelve `Debt::all()` (un marcador), y `DebtController` igualmente consulta `Debt::query()` | Patrón inconsistente: el resto de los módulos usa repositorios concretos |
| 8 | Regla de dominio pendiente | `SpendingRepository`: `// TODO CHECK IF TAG BELONGS TO USER` en la validación de `tag_id` | Posible asignación de una etiqueta de otro espacio |
| 9 | Inconsistencia de datos | Gastos e ingresos se guardan en centavos enteros (`amount` entre 100); las deudas guardan decimales (`500000.00`) | Dos convenciones para el dinero |
| 10 | Rutas y recursos duplicados | Rutas `/spendings/*` y `/earnings/*` junto a `/transactions`; el orden que provoca el 404 de E2; prototipo SPA paralelo en `/prototype` | Mayor superficie de mantenimiento |
| 11 | Internacionalización incompleta y dependencia de CDN | Claves sin traducir (`models.debts`, `fields.type`); `layout.blade.php` depende de Font Awesome Pro, Google Fonts y Chartist externos | La UI se degrada sin Internet |
| 12 | Configuración por defecto frágil | `.env.example` con SMTP a Mailtrap y credenciales vacías; `QUEUE_CONNECTION=database` exige un worker | Falla en el primer uso (E1) |

### 8.3 Mejoras propuestas, en términos de arquitectura en capas

1. **Introducir una capa de servicios (aplicación)** entre controladores y repositorios, de modo que casos de uso como "registrar usuario", "crear gasto con conversión de moneda" o "importar CSV" vivan en un solo lugar y los reutilicen los controladores web y API. Las *Actions* existentes son un buen punto de partida si se amplían.
2. **Eliminar la dependencia de la sesión en las capas inferiores:** pasar `space_id` como parámetro explícito o resolverlo con un *Global Scope* de Eloquent que filtre por espacio, en lugar de repetir `where('space_id', session(...))`.
3. **Transacciones y mensajería asíncrona:** envolver el registro en `DB::transaction` y encolar el correo de verificación (`Mail::queue`) para que un fallo de SMTP no rompa el flujo.
4. **Mover la auditoría a listeners** registrados en `EventServiceProvider`, pasando el usuario como dato del evento en lugar de leer `Auth` en el constructor.
5. **Optimizar consultas:** agrupar `getDailyBalance` en una consulta con `GROUP BY happened_on` y resolver el espacio actual una sola vez por petición.
6. **Aplicar el patrón de interfaz de forma uniforme** (o abandonarlo): si se quiere desacoplar la persistencia, definir contratos para todos los repositorios; si no, retirar la interfaz parcial de deudas.
7. **Limpiar:** eliminar `RefreshSpaceRelations`, unificar rutas, corregir el orden de `/spendings/create`, completar traducciones, unificar la representación del dinero y arreglar las 3 pruebas fallidas.
8. **Evolución futura:** el sistema no justifica hoy microservicios; la evolución natural es un **monolito modular** (módulos Transacciones, Presupuestos, Deudas, Espacios) con fronteras claras, que deje abierta la opción de extraer piezas (por ejemplo, la obtención de tasas de cambio) si el crecimiento lo exige.

### 8.4 Conclusión

Budget es un monolito Laravel funcional y bastante completo que pude instalar y ejecutar localmente sin errores durante la instalación; los problemas aparecieron en tiempo de ejecución y se asocian a decisiones de arquitectura (acoplamiento con SMTP sin transacción, dependencia de la sesión, lógica repartida). Su estructura de carpetas sugiere un modelo en capas, pero sin la disciplina de dependencias que lo haría mantenible a largo plazo. Con una capa de servicios, un acceso a datos desligado de HTTP y una limpieza de las inconsistencias señaladas, la misma aplicación seguiría siendo un monolito, pero con capas realmente separadas.

---

*Evidencias: carpeta [`evidencias/`](evidencias/). Bitácora: [`BITACORA.md`](BITACORA.md). Diagramas: [`diagramas/`](diagramas/).*
