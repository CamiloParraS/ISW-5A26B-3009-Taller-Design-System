# Glosario: palabras que usa el sistema

Vocabulario del código, de la documentación y de la interfaz (español).

## Conceptos del índice

| Término | Significado |
| --- | --- |
| **Raíz (root)** | Carpeta que el usuario eligió indexar. Solo se indexan raíces; nunca se solapan (una raíz dentro de otra se rechaza; agregar una carpeta padre absorbe a las hijas). En la UI: *carpeta*. |
| **Índice** | Base de datos SQLite (`magi.db`) con archivos, fragmentos, texto y vectores. |
| **Archivo (file)** | Fila de `files`. Tiene un `file_id`, un estado y una raíz. |
| **Fragmento (chunk)** | Trozo de texto de un archivo (≤ 400 tokens). Es la unidad que se busca y se vectoriza. |
| **Token** | Unidad del tokenizador del modelo; se usa para medir fragmentos. |
| **Vector / embedding** | Lista de números que representa el significado de un fragmento (texto: 384 dimensiones; imagen: 768). |
| **Miniatura (thumbnail)** | JPEG de 256 px de una imagen o de la primera página de un PDF, en la caché. |
| **Hash de contenido** | `blake3` del archivo; permite saber si cambió de verdad o solo se "tocó". |

## Estados de un archivo

`pending → indexing → indexed | skipped | error`

| Estado | Significado |
| --- | --- |
| `pending` | En cola. |
| `indexing` | Un trabajador lo está procesando. |
| `indexed` | Listo y buscable. |
| `skipped` | No se lee el contenido (tipo no soportado, muy grande, `cloud_only`…); solo se indexa el nombre. |
| `error` | Falló tres veces (reintentos a 30 s y 2 min). |

## Estados de indexación (`IndexState`)

`idle` (al día) · `scanning` (revisando carpetas) · `indexing` (procesando cola) · `paused` (en pausa).

## Estados de una carpeta

| Estado | Significado en la UI |
| --- | --- |
| `ok` | Atento a los cambios. |
| `watch_failed` | No se pueden seguir los cambios; se revisa cada 15 min (sondeo). |
| `permission_denied` | Magi no tiene permiso de lectura. El índice se conserva. |
| `missing` | La carpeta no está (p. ej. unidad desconectada). El índice se conserva. |
| deshabilitada | En pausa: no se busca ni se vigila. |

## Motor y hilos

| Término | Significado |
| --- | --- |
| **Engine** | El motor: hilos, canales y ciclo de vida. |
| **Host** | Supervisor que arranca/reinicia el motor y ofrece la API a la UI. |
| **Planificador (scheduler)** | Cola de trabajo: espera a que un archivo esté estable, deduplica y prioriza (más reciente primero). |
| **Extractor** | Lee un archivo y devuelve fragmentos (`ExtractedDoc`). |
| **Embedder** | Convierte texto o imágenes en vectores (hilo único que posee las sesiones ONNX). |
| **Escritor (writer)** | Único hilo que escribe en la base, una transacción por archivo. |
| **Vigilante (watcher)** | Escucha eventos del sistema de archivos (`notify`). |
| **Reconciliación** | Recorrido completo que compara disco contra índice para detectar cambios perdidos. |
| **Sondeo (poller)** | Reconciliación periódica para carpetas sin vigilante (redes, fallos). |
| **SearchGuard** | Candado de prioridad: mientras hay una búsqueda, la indexación espera entre lotes. |
| **Backfill** | Re-indexar archivos ya listos cuando se activa una función nueva. |
| **Pipeline version** | Número que sube cuando cambia la extracción; fuerza reindexar. |

## Búsqueda

| Término | Significado |
| --- | --- |
| **BM25 / FTS5** | Búsqueda por palabras clave de SQLite. |
| **KNN vectorial** | Búsqueda de los vectores más cercanos (`vec_text`, `vec_image`). |
| **RRF** | *Reciprocal Rank Fusion*: combina varias listas ordenadas en una. |
| **Boost** | Multiplicador pequeño por coincidencia con el nombre o por reciente. |
| **Fuente de coincidencia (match source)** | Por qué salió un resultado. Etiquetas en la UI: **Palabras** (keyword), **Significado** (semantic), **Texto en imagen** (ocr), **Se parece** (visual), **Código QR** (qr), **Nombre** (filename). |
| **Snippet** | Fragmento mostrado con las palabras resaltadas. |

## Funciones de búsqueda (opcionales, ADR-0010)

| Clave | Nombre en la UI | Modelo |
| --- | --- | --- |
| `meaning` | Buscar por significado | multilingual-e5-small |
| `image_text` | Leer texto en imágenes | PaddleOCR PP-OCRv5 |
| `image_visual` | Encontrar imágenes por lo que muestran | SigLIP 2 |

Sin ninguna instalada, la búsqueda es por palabras y nombre.

## Interfaz

| Término | Significado |
| --- | --- |
| **Ventana de búsqueda** | Ventana emergente sin barra de título, tipo Spotlight. Se cierra y se recrea (GPUI no puede ocultarla en Windows). |
| **Configuración** | Ventana normal con barra lateral; hoy solo la sección **Carpetas**. |
| **Bandeja (tray)** | Icono con: Abrir búsqueda, Pausar/Reanudar indexación, Configuración, estado, Salir. |
| **Atajo global (hotkey)** | Combinación configurable que abre/cierra la búsqueda. |
| **`AppEvent`** | Evento que entra al canal único de la app (host, bandeja, atajo, instancia). |
| **`ErrorCode`** | Código de error estable + parámetros; la UI lo traduce. |
| **`Live`** | Estado compartido (estado de indexación, funciones, idioma, tema) que leen las ventanas. |
