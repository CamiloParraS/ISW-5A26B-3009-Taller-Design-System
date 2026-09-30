# Cómo funciona la búsqueda

Búsqueda **híbrida**: palabras clave + significado + imagen, fusionadas en un solo ranking. Todo en el equipo; sin red.

## 1. Antes: qué hay en el índice

Cada archivo de una carpeta elegida se procesa así:

```
archivo ─► extractor ─► fragmentos (≤ 400 tokens, solape 50)
                            ├─► FTS5 (texto para palabras)
                            ├─► vec_text  (e5, 384-d)         si "significado" está activo
                            └─► imágenes: OCR → fragmentos "ocr"; QR → fragmentos "qr";
                                          vec_image (SigLIP 2, 768-d)  si "visual" está activo
```

El nombre y la ruta de cada archivo también son un fragmento, por eso todo archivo se encuentra por nombre.

## 2. En la ventana

1. El usuario escribe. Tras **150 ms** sin teclear se lanza la búsqueda (texto recortado; si no cambió, no se repite).
2. La UI llama a `Host::search` en el executor de fondo; nunca bloquea el hilo de la interfaz.
3. Cada búsqueda lleva un número de **generación**: una respuesta vieja se descarta.

## 3. Algoritmo (`search::hybrid_search`)

1. **Sanear** la consulta para FTS5: se escapan comillas y cada término va entre comillas dobles; el usuario no puede inyectar sintaxis.
2. Ejecutar las listas:
   - **BM25** (FTS5): 100 fragmentos.
   - **Vectorial de texto**: 100 vecinos con `"query: " + consulta` por e5 (omitida si "significado" está apagado).
   - **Vectorial de imagen**: 50 vecinos con la torre de texto de SigLIP; solo se conservan imágenes con coseno ≥ 0,10 (sin ese piso, toda consulta arrastraba las fotos).
3. **Agregar por archivo**: cada archivo queda con el rango de su mejor fragmento en cada lista.
4. **Fusionar con RRF**: `puntaje = Σ peso / (60 + rango)`. Pesos: BM25 1,0 · texto 1,0 · imagen 0,8.
5. **Boosts** (multiplicadores): coincidencia con el nombre (hasta ×1,2) y recencia (hasta ×1,1 si se modificó en los últimos 30 días).
6. **Snippet**: el mejor fragmento. En BM25, resaltado con `snippet()`; en solo-vectorial, los primeros ~200 caracteres. Los de OCR se etiquetan "Texto en imagen" y los QR "Código QR".
7. Devolver los primeros `max_results` archivos (límite fijado entre 1 y 500).

## 4. Resultado (`SearchResult`)

`file_id`, ruta, nombre, tipo, puntaje, snippet con resaltados, página (PDF), miniatura, fecha de modificación y **fuentes de coincidencia**: Palabras · Significado · Texto en imagen · Se parece · Código QR · Nombre.

## 5. Degradación (nunca falla por falta de un modelo)

| Situación | Comportamiento |
| --- | --- |
| Sin función "significado" | Solo palabras + nombre |
| Modelo de imagen ausente o roto | Búsqueda solo de texto |
| Motor reiniciándose | La búsqueda sigue con su propia conexión de solo lectura (palabras/nombre) |
| Aún indexando | Aviso "Puede que falten resultados" |

## 6. Convivencia con la indexación

- La búsqueda usa una **conexión de solo lectura** propia (`query_only`), independiente del escritor. WAL evita bloqueos.
- Toma un **`SearchGuard`**: el embedder revisa el candado entre lotes (≤ 16 fragmentos) y cede, así la búsqueda espera como máximo un lote pequeño.
- Los modelos se cargan al necesitarse y se descargan tras un tiempo inactivo.

## 7. Objetivos

| Métrica | Meta |
| --- | --- |
| Latencia con modelos cargados (100k fragmentos) | p95 ≤ 300 ms |
| Latencia en frío | ≤ 3 s |
| Memoria solo buscando | ≤ 900 MB |

Detalles y mediciones: `docs/SPEC.md` §5.6, `docs/eval.md`, `docs/benchmarks.md`.
