# 🌲 Wood-work — Aserradero Tycoon 3D

Juego **idle / tycoon en 3D** hecho con fines educativos: talas árboles, procesas la madera en un aserradero, surtes una tienda y los clientes compran… pero **por mucho que automatices la producción, sin demanda no vendes**.

Pensado para ilustrar de forma visual conceptos de **automatización de procesos, cuellos de botella, límites de capacidad y demanda/marketing**.

## Jugar

- **En línea (GitHub Pages):** ver la URL de Pages del repo.
- **Local:** abre `index.html` en un navegador moderno (Chrome recomendado). Requiere conexión la primera vez para cargar la librería 3D (Three.js) desde CDN.

## Controles

| Acción | Tecla |
|---|---|
| Mover | `W A S D` o flechas |
| Girar cámara | arrastrar el ratón |
| Zoom | rueda del ratón |
| Interactuar (extra) | `E` / `Espacio` |

Acércate a un 🌲 árbol para talar, al 🏭 aserradero (📥 mete madera / 📤 saca tablas) y a la 🏪 tienda (📥 surte la repisa). Los clientes entran solos según la demanda.

## Mecánicas

- **Cadena de producción visible:** talar → cargar troncos → aserradero (sierra que corta) → tablas → surtir tienda → clientes pagan.
- **Automatización:** herramientas (serrucho → cosechadora), trabajadores NPC (leñadores, carpinteros, vendedores) y **maquinaria** (banda transportadora).
- **Límites en todo:** mochila, capacidad del aserradero y de la repisa — cada mejora amplía el límite.
- **Demanda y marketing:** el precio (elasticidad) y los canales de marketing determinan cuántos clientes llegan. Sobreproducir sin demanda llena la repisa sin generar dinero.

## Tecnología

Un solo archivo HTML con [Three.js](https://threejs.org/) (vía CDN). Sin build, sin dependencias que instalar.

## Archivos

- `index.html` — el juego 3D.
- `clasico-2d.html` — prototipo 2D original (mismas mecánicas económicas, sin 3D).
