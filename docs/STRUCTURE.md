# Project Structure

A complete description of every folder and file in this monorepo.

```
filemd/
├── docs/                         # Project documentation
│   ├── PRD.md                    # Product Requirements Document
│   ├── SRS.md                    # Software Requirements Specification
│   ├── TODO.md                   # List of unfinished work
│   └── STRUCTURE.md              # This file
│
├── backend/                      # Laravel 11 — API only
│   ├── app/
│   │   ├── Http/
│   │   │   ├── Controllers/
│   │   │   │   ├── Controller.php          # Base controller (Laravel default)
│   │   │   │   └── ConvertController.php   # The only controller; handles POST /api/convert
│   │   │   └── Requests/
│   │   │       └── ConvertRequest.php      # File validation: mimes, max 20MB, files[] array
│   │   ├── Services/
│   │   │   ├── ConverterFactory.php        # Resolves the converter based on MIME type
│   │   │   ├── ConvertService.php          # Orchestrator: creates a temp file, calls the converter, deletes the file
│   │   │   └── Converters/
│   │   │       ├── ConverterInterface.php  # Contract: convert(tmpPath, originalName): string
│   │   │       ├── PdfConverter.php        # PDF → Markdown via smalot/pdfparser
│   │   │       ├── DocxConverter.php       # DOCX → Markdown via phpoffice/phpword
│   │   │       ├── XlsxConverter.php       # XLSX → GFM table via phpoffice/phpspreadsheet
│   │   │       ├── PptxConverter.php       # PPTX → slide sections via phpoffice/phppresentation
│   │   │       ├── CsvConverter.php        # CSV → GFM table via native fgetcsv
│   │   │       └── PlainTextConverter.php  # TXT / MD → returned as is
│   │   ├── Models/
│   │   │   └── User.php                    # Laravel default, unused (no auth)
│   │   └── Providers/
│   │       └── AppServiceProvider.php      # Laravel default
│   │
│   ├── bootstrap/
│   │   ├── app.php                         # Registers middleware and service providers
│   │   └── providers.php                   # List of loaded providers
│   │
│   ├── config/
│   │   ├── cors.php                        # CORS: allowed_origins uses the FRONTEND_URL env
│   │   ├── app.php                         # Laravel application configuration
│   │   ├── cache.php                       # Cache driver (file, not actively used)
│   │   ├── logging.php                     # Log channel (single file)
│   │   └── ...                             # Other Laravel config (not modified)
│   │
│   ├── database/
│   │   ├── migrations/                     # Default Laravel migrations (users, cache, jobs)
│   │   │                                   # Unused — this project has no DB
│   │   └── database.sqlite                 # Default SQLite for local testing
│   │
│   ├── public/
│   │   └── index.php                       # Laravel entry point (all requests come through here)
│   │
│   ├── routes/
│   │   ├── api.php                         # POST /api/convert, GET /api/health
│   │   ├── web.php                         # Empty (no Blade views)
│   │   └── console.php                     # Artisan commands (default)
│   │
│   ├── storage/                            # Laravel runtime storage
│   │   ├── framework/                      # Cache, session, compiled views (auto-generated)
│   │   └── logs/                           # Application logs (laravel.log)
│   │
│   ├── tests/
│   │   ├── Feature/
│   │   │   └── ExampleTest.php             # Placeholder — no feature tests yet
│   │   └── Unit/
│   │       └── ExampleTest.php             # Placeholder — no unit tests yet
│   │
│   ├── .env                                # Local environment (not committed)
│   ├── .env.example                        # Env template for a new server
│   ├── artisan                             # Laravel CLI entry point
│   ├── composer.json                       # PHP dependencies
│   └── phpunit.xml                         # PHPUnit configuration
│
├── frontend/                     # React 18 + Vite + TypeScript + Tailwind CSS v4
│   ├── src/
│   │   ├── components/
│   │   │   ├── DropZone.tsx        # Drag & drop area + click to browse; handles DragEvent
│   │   │   ├── FileList.tsx        # List of queued files + "Convert N files" button
│   │   │   ├── FileItem.tsx        # One file row: name, size, status badge, progress bar, delete button
│   │   │   ├── MarkdownOutput.tsx  # Shows the single-file conversion result; raw/preview toggle
│   │   │   └── CopyButton.tsx      # Copies markdown to the clipboard; "Copied!" feedback for 2 seconds
│   │   │
│   │   ├── hooks/
│   │   │   └── useConverter.ts     # All logic: client-side validation, state machine,
│   │   │                           # API fetch, download trigger for multi-file
│   │   │
│   │   ├── types/
│   │   │   └── index.ts            # FileStatus, ConvertFile, ConvertResult
│   │   │
│   │   ├── App.tsx                 # Root component; orchestrates state from useConverter
│   │   ├── main.tsx                # React entry point; mounts to #root
│   │   └── index.css               # Global styles + @import "tailwindcss"
│   │
│   ├── public/
│   │   ├── favicon.svg             # Favicon
│   │   └── icons.svg               # Icon sprite (Vite template default)
│   │
│   ├── dist/                       # Vite build output (auto-generated, not committed)
│   │
│   ├── .env                        # VITE_API_URL for local development
│   ├── .env.example                # Env template
│   ├── index.html                  # HTML shell; Vite injects the bundle here
│   ├── vite.config.ts              # Vite config: plugin react + tailwindcss
│   ├── tsconfig.app.json           # TypeScript config for src/ (strict mode)
│   ├── tsconfig.node.json          # TypeScript config for vite.config.ts
│   └── package.json                # npm dependencies and scripts
│
├── deploy.sh                     # First-time deploy wizard for EC2 / Azure
│                                 # Builds backend + frontend, configures nginx, health check
├── update.sh                     # Pulls the latest from GitHub + rebuild + redeploy
│                                 # Run on the server whenever the code is updated
│
├── README.md                     # Main documentation: usage, deployment, API reference
└── CLAUDE.md                     # Agent instructions: constraints, conventions, TODO
```

---

## Request Flow

```
Browser
  │
  ├─► GET  /               → frontend/dist/index.html  (nginx static)
  │
  └─► POST /api/convert    → nginx → php-fpm → Laravel public/index.php
                                                  │
                                          ConvertRequest (validation)
                                                  │
                                          ConvertController
                                                  │
                                          ConvertService
                                            ├─ tempnam(sys_get_temp_dir())
                                            ├─ ConverterFactory → select converter
                                            ├─ Converter::convert()
                                            └─ finally: unlink(tmpPath)
                                                  │
                                          response()->json(...)
```

---

## Environment Variables

### backend/.env

| Key | Example | Description |
|-----|--------|------------|
| `APP_ENV` | `production` | Laravel mode |
| `APP_KEY` | `base64:...` | Laravel encryption, generate via `artisan key:generate` |
| `APP_URL` | `https://api.yourdomain.com` | Backend URL |
| `FRONTEND_URL` | `https://yourdomain.com` | Used in CORS config |

### frontend/.env

| Key | Example | Description |
|-----|--------|------------|
| `VITE_API_URL` | `https://api.yourdomain.com` | Backend base URL for fetches in the browser |

---

## Files That Do Not Need Changes

The files below are Laravel/Vite boilerplate and are not relevant to the project's functionality:

- `backend/app/Models/User.php` — no auth
- `backend/database/migrations/*` — no DB
- `backend/resources/` — no Blade views
- `backend/routes/web.php` — no web routes
- `frontend/src/assets/` — placeholder images from the Vite template
- `frontend/src/App.css` — old CSS from the template, unused
