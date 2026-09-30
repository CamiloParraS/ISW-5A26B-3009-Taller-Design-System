# Jerarquía de la arquitectura

Magi es **un solo proceso**: la aplicación de escritorio (GPUI) aloja el motor (`magi-core`) dentro de sí misma. No hay servidor, IPC ni webview (ADR-0011).

## Niveles

```
Usuario
  └─ Ventanas de la app (apps/desktop)            ← solo dibujan y reenvían eventos
       ├─ Ventana de búsqueda   (SearchView)
       ├─ Ventana de configuración (SettingsView)
       └─ Bandeja del sistema + atajo global
            │  llamadas directas (bloqueantes, en el executor de fondo de GPUI)
            ▼
       Host (magi-core::host)                     ← supervisa el motor y expone la API
            │
            ▼
       Engine (magi-core::engine)                 ← dueño de hilos y canales
            ├─ Vigilantes (watch)  ─┐
            ├─ Reconciliación       ├─► Planificador ─► Extractores (N) ─► Embedder (1) ─► Escritor (1) ─► SQLite
            └─ Sondeo (poller)     ─┘
            Búsqueda ─► conexión de solo lectura ─► SQLite (FTS5 + sqlite-vec)
```

## Capas y responsabilidades

| Capa | Ubicación | Qué hace | Qué NO hace |
| --- | --- | --- | --- |
| Interfaz | `apps/desktop` | Vistas finas sobre `Host`; bandeja, atajo, instancia única | Lógica de negocio |
| Host | `magi-core/src/host.rs` | Supervisa el motor (lo reinicia si cambian funciones o ajustes) y ofrece la API a la UI | Dibujar nada |
| Engine | `magi-core/src/engine.rs` | Crea hilos/canales y controla el ciclo de vida | Conocer la UI |
| Módulos de dominio | `magi-core/src/*` | `discovery`, `extract`, `chunk`, `embed`, `index`, `watch`, `search`, `db` | Código específico de un SO |
| Plataforma | `magi-core/src/platform/` | Único lugar con `#[cfg(target_os = ...)]` | Lógica de negocio |

## Crates del workspace

| Crate | Rol |
| --- | --- |
| `crates/magi-core` | Toda la lógica. Sin dependencia de UI ni `async` (usa `std::thread` + `crossbeam-channel`) |
| `crates/magi-cli` | CLI para desarrollo y pruebas: `doctor`, `roots`, `index`, `daemon`, `search`, `eval` |
| `apps/desktop` | App GPUI (binario `magi`) |
| `xtask` | Tareas de desarrollo multiplataforma (PDFium, ONNX Runtime, modelos) |

## Módulos de `magi-core`

- `db/` — SQLite (`rusqlite`), modo WAL, FTS5 y `sqlite-vec`. Un único escritor.
- `discovery/` — recorre carpetas y clasifica archivos.
- `extract/` — un extractor por tipo (texto, código, PDF, Office, imágenes).
- `chunk.rs` — parte el texto en fragmentos de hasta 400 tokens con 50 de solapamiento.
- `embed/` — embedders de texto (e5) e imagen (SigLIP 2) y gestor de modelos (carga perezosa, descarga por inactividad).
- `index/` — planificador → pipeline → hilo escritor.
- `watch/` — vigilancia del sistema de archivos, reconciliación y sondeo.
- `search/` — BM25 + vectorial + fusión RRF.
- `dto.rs` — los tipos que ve la UI.

## Reglas de dependencia

1. La UI depende de `Host`; `Host` depende de `Engine`; nunca al revés.
2. La UI abre o revela archivos **solo por `file_id`** a través de `Host`; nunca construye rutas.
3. Los datos viajan a la UI como tipos Rust (`dto.rs`); el texto que ve el usuario se traduce en la UI (`i18n/`).
4. Nada se escribe en las carpetas del usuario; solo se leen.
