# Navegación

Magi no tiene una ventana principal única: vive en la bandeja y se navega entre **dos ventanas** y un menú.

## Mapa

```
                 ┌──────────────── Arranque (`magi`) ────────────────┐
                 │  lanzamiento normal      → abre Configuración      │
                 │  `magi --toggle`         → alterna Búsqueda        │
                 │  segundo lanzamiento     → "show" a la instancia   │
                 └───────────────────────────────────────────────────┘
                                        │
        ┌───────────────────────────────┼───────────────────────────────┐
        ▼                               ▼                               ▼
  Atajo global                    Icono de bandeja                 Configuración
  (alterna Búsqueda)              ├─ Abrir búsqueda ──► Búsqueda   └─ Carpetas
                                  ├─ Pausar / Reanudar indexación      (agregar, quitar,
                                  ├─ Configuración ──► Configuración    ver estado)
                                  ├─ (línea de estado)
                                  └─ Salir (cierra el motor y la app)
```

## Ventanas

### Búsqueda
- Emergente, sin barra de título, 680 px de ancho, no movible ni redimensionable.
- Se abre en el monitor donde está el cursor, con solo la fila de entrada (58 px); crece con los resultados hasta 560 px.
- Puede tener fondo translúcido (según `ui.transparency_mode`).
- **Alternar** (atajo o `--toggle`): si está abierta la cierra; si no, la abre. Desde la bandeja, *Abrir búsqueda* la trae al frente o la abre.
- Cerrarla **nunca** cierra la aplicación.

### Configuración
- Ventana normal, sólida, centrada. Si ya existe, se trae al frente en vez de duplicarse.
- Barra lateral con una sección (**Carpetas**); otras secciones llegarán después.

## Teclado (ventana de búsqueda)

| Tecla | Acción |
| --- | --- |
| Escribir | Busca (espera 150 ms sin teclear) |
| `↑` / `↓` | Mover la selección |
| `Enter` | Abrir el archivo con la app predeterminada |
| `Ctrl/Cmd + Enter` | Mostrar el archivo en su carpeta |
| `Ctrl/Cmd + C` | Copiar la ruta (sin texto seleccionado en la entrada) |
| `Esc` | Cerrar la ventana |
| Clic en resultado | Selecciona |

Toda la búsqueda es operable solo con teclado (NFR-10).

## Cómo se procesa una acción

Toda señal externa (bandeja, atajo, socket de instancia única, eventos del `Host`) se convierte en un `AppEvent` en **un único canal**. Una tarea en el hilo principal lo lee y llama a `Shell::handle`:

| `AppEvent` | Efecto |
| --- | --- |
| `Toggle` | Cierra la búsqueda si existe; si no, la abre |
| `Settings` / bandeja *Configuración* | Trae al frente o abre Configuración |
| bandeja *Abrir búsqueda* | Trae al frente o abre Búsqueda |
| bandeja *Pausar/Reanudar* | Llama al motor en segundo plano |
| bandeja *Salir* | Cierra la búsqueda, detiene el motor y sale |
| `Host(Status)` | Actualiza bandeja y estado compartido (`Live`) |
| `Host(Features)` | Actualiza la lista de funciones (descargas, backfill) |

## Instancia única

Solo corre una instancia. Un socket local (tubería con nombre en Windows; nunca TCP) `magi-<usuario>.sock` recibe `toggle` o `show` desde otros lanzamientos.

## Sin bandeja o sin atajo

Si el icono de bandeja o el atajo no se pueden registrar, se registra una advertencia y la app sigue usable: un lanzamiento normal abre Configuración y `magi --toggle` abre la búsqueda.

## Estados que se ven al navegar

- Sin escribir: una introducción con el número de archivos, y un aviso si aún se indexa.
- Sin resultados: "Ningún archivo coincide con “…”".
- Error de búsqueda: mensaje neutro con botón **Reintentar**.
- Función desactivada: oferta *Activar (tamaño)*, progreso de descarga y de actualización.
