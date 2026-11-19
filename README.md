# Taller 14 — Instalación y análisis de una aplicación monolítica (Budget, Laravel)

**Estudiante:** Luisa Fernanda Velandia Salamanca · **Código:** U00179371
**Universidad Autónoma de Bucaramanga (UNAB)** · Arquitectura de Software · noviembre de 2026
**Docentes:** MSc. Fabian Suarez, MSc. Javier Pinzon, MSc. Feisar Moreno

## Resumen

Informe técnico de instalación y análisis de **Budget**, una aplicación de finanzas personales hecha con Laravel 10, Livewire 2 y Vite/Vue/Tailwind. Instalé la aplicación en Windows 11 (PHP 8.2 y MariaDB de XAMPP), la ejecuté localmente, probé sus funcionalidades con un navegador real, ejecuté su suite de pruebas (92 pruebas, 3 fallos preexistentes), analicé su arquitectura a partir del código y la modelé con C4 (niveles 1 a 3). Documenté los errores reales que aparecieron (SMTP en el registro, orden de rutas, base de pruebas sin migrar) y las consultas a la IA usadas para resolverlos.

- Aplicación original (no incluida aquí): <https://github.com/fmoreno4/butget_laravel> (autor: fmoreno4), commit analizado `c9d1f3e`.
- Este repositorio contiene solo mi documentación y evidencias.

## Índice del informe

Documento completo: [`docs/informe/Informe-Taller14-LuisaVelandia.pdf`](docs/informe/Informe-Taller14-LuisaVelandia.pdf) · [`.docx`](docs/informe/Informe-Taller14-LuisaVelandia.docx) · versión Markdown: [`docs/INFORME.md`](docs/INFORME.md)

1. [Introducción](docs/INFORME.md#1-introducción)
2. [Preparación del entorno](docs/INFORME.md#2-preparación-del-entorno)
3. [Proceso de instalación](docs/INFORME.md#3-proceso-de-instalación)
4. [Registro de errores y soluciones](docs/INFORME.md#4-registro-de-errores-y-soluciones) (incluye transparencia sobre el uso de IA)
5. [Ejecución del sistema](docs/INFORME.md#5-ejecución-del-sistema)
6. [Análisis arquitectónico](docs/INFORME.md#6-análisis-arquitectónico)
7. [Modelo C4](docs/INFORME.md#7-modelo-c4)
8. [Análisis crítico](docs/INFORME.md#8-análisis-crítico)

Otros archivos: [`docs/BITACORA.md`](docs/BITACORA.md) (bitácora cronológica), [`docs/evidencias/`](docs/evidencias) (capturas), [`docs/diagramas/`](docs/diagramas) (fuentes Mermaid y exportaciones PNG/SVG).

## Conclusión en pocas líneas

Budget es un **monolito Laravel** (un solo despliegue, una sola base de datos, vistas Blade con Vue y Livewire incrustados) con una separación por capas **parcial**: existe la convención controlador, repositorio, modelo, pero los repositorios dependen de la sesión HTTP, parte de la lógica vive en controladores y modelos, y la abstracción por interfaces solo se aplicó al módulo de deudas. El informe propone una capa de servicios, transacciones, un alcance por espacio desacoplado de HTTP y un monolito modular como evolución.

## Diagramas C4

### Nivel 1 — Contexto

```mermaid
flowchart TB
    U(["<b>Usuario</b><br/>[Persona]<br/>Registra ingresos, gastos,<br/>presupuestos y deudas"])
    API(["<b>Cliente de la API</b><br/>[Persona / software]<br/>Usa /api/* con cabecera api-key"])

    S["<b>Budget</b><br/>[Sistema de software]<br/>Aplicación web monolítica Laravel<br/>para el control de finanzas personales"]

    SMTP["<b>Servidor SMTP</b><br/>[Sistema externo]<br/>Correo de verificación, reseteo<br/>de contraseña e informe semanal"]
    FX["<b>currency-api (jsDelivr)</b><br/>[Sistema externo]<br/>Tasas de conversión de monedas"]
    CDN["<b>CDN de recursos web</b><br/>[Sistema externo]<br/>Font Awesome, Google Fonts, Chartist"]

    U -->|"Usa mediante navegador<br/>[HTTP/HTML]"| S
    API -->|"Consulta y crea transacciones<br/>[HTTP/JSON]"| S
    S -->|"Envía correos<br/>[SMTP]"| SMTP
    S -->|"Descarga tasas diarias<br/>[HTTPS/JSON]"| FX
    U -.->|"El navegador descarga<br/>fuentes, iconos y gráficos"| CDN

    classDef persona fill:#08427b,stroke:#052e56,color:#fff
    classDef sistema fill:#1168bd,stroke:#0b4884,color:#fff
    classDef externo fill:#999,stroke:#6b6b6b,color:#fff
    class U,API persona
    class S sistema
    class SMTP,FX,CDN externo
```

### Nivel 2 — Contenedores

```mermaid
flowchart TB
    U(["<b>Usuario</b><br/>[Persona]"])
    API(["<b>Cliente de la API</b><br/>[Software]"])

    subgraph SIS["Sistema Budget (unidad de despliegue única)"]
        direction TB
        NAV["<b>Navegador web</b><br/>[Contenedor: JavaScript]<br/>HTML de Blade + Vue 2 (TransactionWizard,<br/>DatePicker, ColorPicker) + Livewire.js + Chartist"]
        APP["<b>Aplicación monolítica Laravel</b><br/>[Contenedor: PHP 8.2, Laravel 10, Livewire 2]<br/>Rutas, middlewares, controladores,<br/>repositorios, acciones, modelos Eloquent, vistas Blade"]
        JOB["<b>Planificador y cola</b><br/>[Contenedor: php artisan schedule:run / queue:work]<br/>ProcessRecurrings, FetchConversionRates,<br/>SendWeeklyReports (mismo código base)"]
        DB[("<b>Base de datos</b><br/>[Contenedor: MySQL 8 (dump) · MariaDB 10.4 (local)]<br/>23 tablas: users, spaces, spendings,<br/>earnings, tags, budgets, debts, jobs...")]
        FS[("<b>Almacenamiento de archivos</b><br/>[Sistema de archivos: storage/app]<br/>CSV importados, avatares, adjuntos")]
    end

    SMTP["<b>Servidor SMTP</b><br/>[Externo]"]
    FX["<b>currency-api (jsDelivr)</b><br/>[Externo]"]
    CDN["<b>CDN de recursos web</b><br/>[Externo]"]

    U -->|"Interactúa<br/>[HTTP]"| NAV
    NAV -->|"Peticiones GET/POST, Axios y<br/>/livewire/message [HTTP, sesión + CSRF]"| APP
    API -->|"/api/* [HTTP/JSON, api-key]"| APP
    APP -->|"Lee y escribe<br/>[PDO / Eloquent]"| DB
    APP -->|"Guarda y lee archivos<br/>[Storage]"| FS
    APP -->|"Encola jobs<br/>[tabla jobs]"| DB
    JOB -->|"Lee la cola y los datos<br/>[PDO]"| DB
    JOB -->|"Envía correos<br/>[SMTP]"| SMTP
    JOB -->|"GET tasas<br/>[HTTPS]"| FX
    APP -->|"Correo de verificación y reseteo<br/>[SMTP]"| SMTP
    NAV -.->|"Carga recursos"| CDN

    classDef persona fill:#08427b,stroke:#052e56,color:#fff
    classDef cont fill:#438dd5,stroke:#2e6295,color:#fff
    classDef externo fill:#999,stroke:#6b6b6b,color:#fff
    class U,API persona
    class NAV,APP,JOB,DB,FS cont
    class SMTP,FX,CDN externo
```

### Nivel 3 — Componentes

```mermaid
flowchart TB
    NAV(["Navegador / Cliente API"])

    subgraph APP["Contenedor: Aplicación monolítica Laravel (app/, routes/, resources/)"]
        direction TB
        RT["<b>Rutas</b><br/>routes/web.php (92 rutas en total)<br/>routes/api.php"]

        subgraph MW["Middlewares (app/Http/Middleware, Kernel)"]
            direction LR
            M1["Grupo web: EncryptCookies,<br/>StartSession, VerifyCsrfToken"]
            M2["auth / guest<br/>RedirectIfAuthenticated"]
            M3["SetLocale<br/>RefreshSpaceRelations"]
            M4["ResolveApiKey<br/>(solo /api)"]
        end

        subgraph PRES["Capa de presentación"]
            direction LR
            CW["<b>Controladores web</b> (app/Http/Controllers)<br/>Dashboard, Transaction, Spending, Earning,<br/>Tag, Budget, Debt, Report, Import, Settings, Space"]
            CA["<b>Controladores API</b><br/>app/Http/Controllers/Api"]
            LW["<b>Componentes Livewire</b><br/>WidgetList, WidgetWizard"]
            V["<b>Vistas</b><br/>resources/views (Blade, 70 archivos)<br/>resources/assets/js (Vue 2)"]
        end

        subgraph NEG["Lógica de aplicación y negocio"]
            direction LR
            FR["<b>Form Requests y Policies</b><br/>DebtRequest, DebtDetailRequest,<br/>SpendingPolicy, TagPolicy..."]
            ACT["<b>Actions</b> (9)<br/>CreateUserAction, CreateSpaceAction..."]
            REP["<b>Repositorios</b> (15)<br/>SpendingRepository, BudgetRepository,<br/>DashboardRepository, ORMDebtRepository"]
            EV["<b>Eventos</b> (8)<br/>TransactionCreated, TagCreated...<br/>registran Activity"]
            JB["<b>Jobs y Mail</b><br/>ProcessRecurrings, FetchConversionRates,<br/>SendWeeklyReports, VerifyRegistration"]
        end

        subgraph DAT["Capa de datos"]
            MO["<b>Modelos Eloquent</b> (18)<br/>User, Space, Spending, Earning, Tag,<br/>Budget, Debt, Widget, Activity..."]
        end
    end

    DB[("MySQL / MariaDB")]
    EXT["SMTP y currency-api"]

    NAV -->|HTTP| RT
    RT --> MW
    MW --> CW
    MW --> CA
    MW --> LW
    CW --> V
    LW --> V
    CW --> FR
    CW --> REP
    CW --> ACT
    CA --> REP
    LW --> MO
    REP --> MO
    ACT --> MO
    CW -.->|"consultas directas<br/>(Tag::query, Debt::query)"| MO
    MO -->|"dispatchesEvents"| EV
    EV --> MO
    JB --> REP
    JB --> MO
    MO --> DB
    JB --> EXT

    classDef comp fill:#85bbf0,stroke:#5d82a8,color:#000
    classDef ext fill:#999,stroke:#6b6b6b,color:#fff
    class RT,M1,M2,M3,M4,CW,CA,LW,V,FR,ACT,REP,EV,JB,MO comp
    class DB,EXT ext
```

Versiones exportadas: [`nivel 1`](docs/diagramas/c4-nivel1-contexto.png), [`nivel 2`](docs/diagramas/c4-nivel2-contenedores.png), [`nivel 3`](docs/diagramas/c4-nivel3-componentes.png).

## Cómo reproducir la instalación

Requisitos: PHP 8.1 o superior con la extensión `calendar` (además de `pdo_mysql`, `mbstring`, `openssl`, `intl`, `zip`, `fileinfo`), Composer 2, Node.js con npm y MySQL o MariaDB. Comandos en Git Bash, uno por línea:

```bash
git clone https://github.com/fmoreno4/butget_laravel.git
cd butget_laravel
composer install
cp .env.example .env
php artisan key:generate
```

Crea una base de datos vacía (por ejemplo `budget_taller14`) y en el `.env` ajusta `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` y, para desarrollo sin correo real, agrega `MAIL_MAILER=log` (evita el error 530 de SMTP descrito en el informe). Luego:

```bash
php artisan migrate
npm install
npm run build
php artisan storage:link
php artisan serve --host=127.0.0.1 --port=8000
```

Abre <http://127.0.0.1:8000> y usa "Registrarse" para crear tu usuario. Para las pruebas, crea una base `<DB_DATABASE>_testing`, ejecuta `php artisan migrate --database=testing --force` y luego `php vendor/bin/phpunit`.
