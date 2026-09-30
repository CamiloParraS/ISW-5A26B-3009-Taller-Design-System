# Documentación de arquitectura de Magi

Documentos en español que resumen cómo está construido Magi (búsqueda semántica local de archivos). La fuente de verdad sigue siendo `docs/SPEC.md`; si algo difiere, manda la especificación.

| Documento | Contenido |
| --- | --- |
| [01-jerarquia.md](01-jerarquia.md) | Niveles del sistema (UI → Host → Engine → base de datos), responsabilidades de cada capa, crates, módulos y reglas de dependencia |
| [02-glosario.md](02-glosario.md) | Vocabulario: raíz, fragmento, estados de archivo/carpeta, hilos del motor, términos de búsqueda, funciones opcionales y de interfaz |
| [03-navegacion.md](03-navegacion.md) | Ventanas (búsqueda y configuración), bandeja, atajo global, instancia única, teclado y flujo de eventos |
| [04-busqueda.md](04-busqueda.md) | Qué se indexa, algoritmo híbrido (BM25 + vectores + RRF + boosts), degradación, convivencia con la indexación y metas de rendimiento |
