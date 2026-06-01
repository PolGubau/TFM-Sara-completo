# Frontend — Explorador del Grafo Fotográfico Areñas

## Referencia de diseño

El cliente ha indicado como referencia **"La red del exilio literario"** (gexel.graphery.com):
- Layout split: **grafo a la izquierda (~75%) + panel de detalle a la derecha (~25%) siempre visible**
- El grafo es la **vista única y principal** — no hay galería separada
- Click en un nodo → panel derecho muestra información y relaciones organizadas por tipo
- Dropdown para filtrar qué tipo de nodo mostrar

Nuestra adaptación: los nodos imagen **muestran la foto como thumbnail** dentro del grafo (nodo circular con imagen), y el panel derecho muestra la foto ampliada + conexiones agrupadas por dimensión.

---

## Análisis del producto

### Datos disponibles
- **GEXF** (`output/grafo_final/grafo_multidimensional.gexf`) con ~1400 aristas y 3 tipos de nodos:
  - `group=1` **Imagen** (lightblue) — cada foto analizada del fondo Areñas
  - `group=2` **Palabra** (lightgreen) — palabras extraídas por HTR del texto manuscrito
  - `group=3` **Hub-Año** (dark) — nodo central por año predicho (`year_node_1921`, etc.)
- **Relaciones**: `contiene_palabra` · `pertenece_a_año` · `mismo_año`
- **Imágenes físicas**: en `input/fotos/` (fondo Areñas)
- **CSV de soporte**: 312 fotos con caption, topónimos, nombres propios y año real

### El problema de UX
Gephi es potente pero opaco para investigadores de patrimonio no técnicos. La web debe:
1. Poner el **grafo como protagonista** — es la metáfora central del TFM
2. Mostrar las fotos **dentro** del grafo, no como grid aparte
3. Organizar las conexiones de cada foto por dimensión semántica en el panel lateral

---

## Layout UI — Split screen (referencia directa)

```
┌─────────────────────────────────────────┬──────────────────────────┐
│                                         │  Panel de detalle        │
│   GRAFO INTERACTIVO (Sigma.js WebGL)    │  ─────────────────────   │
│                                         │  [Foto ampliada]         │
│   ○─────●─────○                         │  Filename / año          │
│   │  [img]    │                         │  Año predicho: 1921      │
│   ○      ──── ●─────○                   │                          │
│      [img]  [img]                       │  ── mismo_año ──         │
│                                         │  [t] [t] [t]             │
│   [Filtro dimensión ▾]  [año][palabras] │                          │
│                                         │  ── contiene_palabra ──  │
└─────────────────────────────────────────│  • joan  • vida  • el    │
                                          └──────────────────────────┘
```

Sin estado vacío: al cargar se pre-selecciona el nodo imagen más conectado.

---

## Stack técnico recomendado

| Capa          | Tecnología                        | Justificación                                                          |
| ------------- | --------------------------------- | ---------------------------------------------------------------------- |
| Framework     | **Next.js 14** (App Router)       | SSG para el grafo pre-procesado; API route para imágenes locales       |
| Grafo         | **Sigma.js v3** + **Graphology**  | GEXF nativo, WebGL, React bindings (`@react-sigma/core`), nodos custom |
| Nodos imagen  | **Sigma custom node renderer**    | Renderiza thumbnails circulares dentro del canvas WebGL                |
| Estilos       | **Tailwind CSS v4**               | Sin runtime CSS; ideal para panel lateral con dark mode                |
| Animaciones   | **Framer Motion**                 | Transición del panel al cambiar nodo seleccionado                      |
| Estado global | **Zustand**                       | Store con: `selectedNode`, `activeLayers`, `yearRange`                 |
| Parsing GEXF  | **graphology-gexf**               | Parser oficial; genera el grafo Graphology desde el XML                |
| Layout        | **graphology-layout-forceatlas2** | Pre-calculado en build → coordenadas `x,y` fijas en JSON               |
| Tipografía    | **Inter**                         | Académica, legible, consistente con la referencia                      |

---

## Procesado de datos (build-time)

El GEXF se convierte a JSON en `scripts/parse-gexf.ts` durante el build. Nunca se parsea en el cliente.

```
scripts/parse-gexf.ts
  → public/data/graph.json        nodos + aristas + coordenadas x,y (ForceAtlas2 pre-calculado)
  → public/data/images-index.json map filename → { caption, year, url }
```

Estructura de `graph.json`:
```json
{
  "nodes": [
    { "id": "IAAH_GUDIOL_34303.jpg", "type": "image", "year": 1921,
      "words": ["joan", "vida"], "x": 120.4, "y": -88.1 },
    { "id": "year_node_1921", "type": "year", "label": "1921", "x": 130.0, "y": -90.0 },
    { "id": "joan", "type": "word", "freq": 5, "x": 115.0, "y": -82.0 }
  ],
  "edges": [
    { "source": "IAAH_GUDIOL_34303.jpg", "target": "year_node_1921", "relation": "pertenece_a_año" },
    { "source": "IAAH_GUDIOL_34303.jpg", "target": "joan", "relation": "contiene_palabra" }
  ]
}
```

---

## Estructura de componentes

```
app/
├── page.tsx                         ← Shell: split layout (graph + panel)
├── components/
│   ├── graph/
│   │   ├── GraphCanvas.tsx          ← SigmaContainer con nodos custom
│   │   ├── ImageNodeRenderer.tsx    ← Renderer WebGL: thumbnail circular por nodo imagen
│   │   ├── GraphControls.tsx        ← Zoom in/out, reset view, botón fullscreen
│   │   └── LayerFilter.tsx          ← Dropdown: todas / solo fotos / solo año / solo palabras
│   └── panel/
│       ├── DetailPanel.tsx          ← Panel derecho siempre visible
│       ├── PhotoPreview.tsx         ← Imagen grande + metadatos
│       ├── ConnectionSection.tsx    ← Sección reutilizable por tipo de relación
│       └── RelatedThumb.tsx         ← Miniatura clickable de foto relacionada
├── hooks/
│   ├── useGraphData.ts              ← Fetch + carga de graph.json
│   ├── useEgoGraph.ts               ← Extrae vecinos del nodo seleccionado
│   └── useLayerFilter.ts            ← Computa nodeReducer / edgeReducer para Sigma
└── store/
    └── graphStore.ts                ← Zustand: selectedNodeId, activeLayers, hoveredNodeId
```

---

## Interacciones clave (patrón referencia)

### Flujo principal
1. App carga → grafo renderizado con thumbnails → panel pre-selecciona nodo más conectado
2. Usuario hace pan/zoom en el grafo libremente
3. **Click en nodo imagen** → `selectedNodeId` en Zustand → panel actualiza con Framer Motion crossfade
4. Panel muestra: foto ampliada · año predicho · sección `mismo_año` con thumbs clickables · sección `contiene_palabra` con chips de palabras
5. Click en una miniatura del panel → selecciona ese nodo (el grafo hace `sigma.getCamera().animate()` hacia él)

### Click en nodos no-imagen
- **Nodo año** (`year_node_1921`): panel lista todas las fotos de ese año como thumbs clickables
- **Nodo palabra** (`joan`): panel lista todas las fotos que contienen esa palabra; el grafo aplica highlight a sus aristas

### Filtrado de capas (dropdown referencia)
```
Todas las dimensiones  ←→  Solo fotografías  ←→  Solo año  ←→  Solo palabras
```
Implementado como `nodeReducer` + `edgeReducer` en Sigma: nodos no activos se renderizan con `color: transparent` y `hidden: true` (no eliminados del grafo, para mantener layout estable).

---

## Implementación por fases

### Fase 1 — Datos (1 día)
- [ ] `scripts/parse-gexf.ts`: GEXF → JSON + ForceAtlas2 → coordenadas persistidas
- [ ] Copiar `input/fotos/` → `public/fotos/` en build script
- [ ] Setup Next.js 14 + Tailwind + Zustand

### Fase 2 — Grafo base (2-3 días)
- [ ] `GraphCanvas` con `@react-sigma/core` + carga de `graph.json`
- [ ] Nodos estándar funcionales (color por tipo, label visible)
- [ ] Click en nodo → actualiza Zustand `selectedNodeId`
- [ ] `LayerFilter` dropdown con `nodeReducer` / `edgeReducer`

### Fase 3 — Thumbnails en nodos (1-2 días)
- [ ] `ImageNodeRenderer`: custom WebGL renderer que pinta `<canvas>` circular con la foto
- [ ] Cache de imágenes pre-cargadas en `Map<nodeId, HTMLImageElement>` antes de renderizar
- [ ] Fallback a nodo color sólido si la imagen no está disponible

### Fase 4 — Panel lateral (2 días)
- [ ] `DetailPanel` siempre visible; `ConnectionSection` por tipo de relación
- [ ] `useEgoGraph` extrae vecinos por tipo de arista del grafo Graphology
- [ ] Animación Framer Motion crossfade al cambiar `selectedNodeId`
- [ ] Click en thumb del panel → `sigma.getCamera().animate()` hacia ese nodo

### Fase 5 — Pulido (1 día)
- [ ] Hover sobre nodo → highlight ego-graph (vecinos opacos, resto desvanecido)
- [ ] Tooltip con nombre al hacer hover
- [ ] Responsive mínimo (panel colapsa en móvil con botón toggle)

---

## Notas de implementación críticas

**Thumbnails en WebGL**: Sigma.js permite registrar un `NodeProgramImage` oficial que pinta texturas. Usar `@sigma/node-image` (paquete oficial del ecosistema) en lugar de un renderer custom desde cero.

**Pre-carga de imágenes**: Las texturas WebGL deben estar cargadas antes de que Sigma renderice. Usar `Promise.all` sobre todos los nodos imagen antes de montar `SigmaContainer`.

**Posiciones fijas**: ForceAtlas2 se ejecuta una sola vez en `parse-gexf.ts` con `graphology-layout-forceatlas2` en modo iteraciones (500 iter). Las coordenadas `x,y` se guardan en `graph.json`. En el cliente NO hay re-layout — el grafo es estático en posición.

**Mapping imagen → archivo**: El `id` del nodo imagen en el GEXF ES el filename (`IAAH_GUDIOL_34303.jpg`). La ruta de la foto es `public/fotos/{id}`. No hay mapeo adicional necesario.

**Highlight sin eliminar nodos**: Nunca usar `graph.dropNode()` para filtrar — rompe el layout. Usar siempre `nodeReducer` / `edgeReducer` para cambiar opacidad/visibilidad sin mutar el grafo subyacente.
