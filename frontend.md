# Frontend — Explorador del Fondo Fotográfico Areñas
<!-- Documento autocontenido. Todo lo necesario para construir el frontend está aquí. -->
<!-- No se requiere acceder a ningún otro fichero para entender el proyecto.            -->


## Contexto del proyecto

Este es el frontend de un **TFM de humanidades digitales** sobre el Fondo Fotográfico Areñas: una colección de fotografías catalanas de principios del siglo XX (1909–1935) analizadas con modelos de visión, HTR y detección de vestimenta.

El resultado del pipeline de análisis es un **grafo multidimensional** que conecta fotografías con:
- los **años** en que fueron tomadas (predicción de modelo)
- las **prendas de vestimenta** que aparecen en cada foto (detección automática)
- las **palabras** transcritas del texto manuscrito que las acompaña (HTR)

El frontend es un **explorador interactivo** de ese grafo. La referencia de diseño es ["La red del exilio literario"](https://gexel.graphery.com): layout split (grafo + panel lateral), el grafo como protagonista único, sin galería separada.

**Nuestra adaptación**: los nodos imagen muestran la foto como thumbnail circular dentro del grafo WebGL; el panel lateral muestra la foto ampliada + sus conexiones agrupadas por tipo (vestimenta, año, palabras HTR).

---

## Datos del grafo (fuente de verdad)

### Fichero
`grafo_multidimensional.gexf` — se entrega junto a este documento.

- Formato: GEXF 1.2draft (`xmlns="http://www.gexf.net/1.2draft"`)
- **740 nodos · 4120 aristas**

### Nodos — 4 dimensiones

| `dimension`     | Cantidad | `node_type` | `group` | Descripción                               |
| --------------- | -------- | ----------- | ------- | ----------------------------------------- |
| `imagen`        | 564      | `foto`      | `1`     | Cada fotografía del fondo                 |
| `transcripcion` | 127      | `palabra`   | `2`     | Palabras extraídas por HTR del manuscrito |
| `año`           | 17       | `fecha`     | `3`     | Nodo hub por año (`year_node_1921`, etc.) |
| `vestimenta`    | 32       | —           | —       | Prenda de ropa detectada en la foto       |

**Años cubiertos**: 1909, 1919, 1921, 1922, 1923, 1924, 1925, 1926, 1927, 1928, 1929, 1930, 1931, 1932, 1933, 1934, 1935 (IDs: `year_node_1909`, `year_node_1919`, …)

**Prendas (32)**: baroque coat · bow tie · cassock · cloak · crinoline dress · evening gown · fan · flamenco dress · folk costume · gloves · handbag · historical costume · laurel crown · mantilla · medieval costume · military uniform · morning coat · naval uniform · nun habit · overcoat · renaissance dress · sailor suit · shawl · tailcoat · top hat · traditional costume · tuxedo · turban · veil · walking cane *(+2 más)*

### Atributos de nodo en el GEXF

Los `<attvalue>` usan IDs numéricos (no nombres). Mapa completo:

| ID numérico | Nombre          | Tipo   | Presente en              | Descripción                                              |
| ----------- | --------------- | ------ | ------------------------ | -------------------------------------------------------- |
| `0`         | `title`         | string | todos                    | igual al `label`                                         |
| `1`         | `node_type`     | string | todos                    | `foto` / `palabra` / `fecha`                             |
| `2`         | `pk`            | string | todos                    | clave natural (= `id`)                                   |
| `3`         | `group`         | long   | imagen/transcripcion/año | `1` / `2` / `3`                                          |
| `4`         | `size`          | double | todos                    | tamaño sugerido por el pipeline                          |
| `5`         | `color`         | string | imagen/transcripcion/año | color hex del pipeline (sólo orientativo)                |
| `6`         | `dimension`     | string | todos                    | `imagen` / `transcripcion` / `año` / `vestimenta`        |
| `7`         | `year`          | long   | imagen                   | año predicho por el modelo                               |
| `8`         | `confianza`     | double | imagen                   | confianza de la predicción 0–100                         |
| `9`         | `type`          | string | imagen/año               | forma: `circle` (foto) o `square` (año)                  |
| `10`        | `community`     | long   | todos                    | comunidad Louvain 0–11                                   |
| `11`        | `degree_static` | long   | todos                    | grado del nodo (nº de aristas) — usar para tamaño visual |
| `12`        | `n_fotos`       | long   | año                      | número de fotos asociadas a ese año                      |

> `graphology-gexf` resuelve los IDs numéricos automáticamente al parsear. No hace falta mapearlo manualmente en el cliente.

### Aristas — 4 tipos de relación

| `relation`         | Cantidad | `dimension`     | Conecta                                      |
| ------------------ | -------- | --------------- | -------------------------------------------- |
| `lleva_puesto`     | 2 007    | `vestimenta`    | foto → prenda                                |
| `mismo_año`        | 1 596    | `año`           | foto ↔ foto (mismo año; OCULTAR por defecto) |
| `pertenece_a_año`  | 361      | `año`           | foto → `year_node_XXXX`                      |
| `contiene_palabra` | 156      | `transcripcion` | foto → palabra HTR                           |

Atributos de arista: `relation` (string, ID `13`) + `dimension` (string, ID `14`). Sin `weight`.

> **⚠️ `mismo_año`**: Las 1 596 aristas foto↔foto son necesarias para que ForceAtlas2 agrupe las fotos por época en el layout, pero son ruido visual puro. Se incluyen en el grafo JSON para el layout y se ocultan con `edgeReducer` en el cliente por defecto.

### CSV de metadatos (`input.csv`)

312 filas. Columnas:

| Columna       | Descripción                                                              |
| ------------- | ------------------------------------------------------------------------ |
| `filename`    | **Clave de unión** con el `id` del nodo imagen en el GEXF                |
| `nom_arxiu`   | Nombre del archivo histórico de origen                                   |
| `nom_fons`    | Nombre del fondo documental (ej: "Anton Moix", "IAAH Gudiol")            |
| `caption`     | Descripción de la foto en catalán (texto libre, hasta ~300 chars)        |
| `toponims`    | Lugar geográfico (puede estar vacío)                                     |
| `noms_propis` | Personas mencionadas (puede estar vacío)                                 |
| `url`         | **URL pública directa** de la imagen en alta resolución                  |
| `year`        | Año del CSV (puede diferir del `year` predicho por el modelo en el GEXF) |

URL de ejemplo:
`https://static.arxiusenlinia.cultura.gencat.cat/active/GIAC/NOTEXTUAL/310/122/1/1149/ACCB_681862_310012230001149.jpg`

> Las fotos están en `arxiusenlinia.cultura.gencat.cat`. Son de acceso público. Verificar CORS antes de usar como textura en el canvas WebGL de Sigma (ver sección "Notas críticas").

---

## Layout UI

### Split screen (escritorio, ≥1024px)

```
┌──────────────────────────────────────────────┬────────────────────────────┐
│  [logo / título]   [■ imagen][■ año]          │                            │
│                    [■ vestimenta][■ palabras] │  PANEL LATERAL             │
├──────────────────────────────────────────────┤  (siempre visible,         │
│                                              │   scroll interno)          │
│   GRAFO INTERACTIVO (WebGL, Sigma.js)        │  ─────────────────────     │
│                                              │  ┌─────────────────────┐   │
│   ●[img]──────●[img]                         │  │  [foto 1:1]         │   │
│        ╲     ╱                               │  └─────────────────────┘   │
│    [■ año 1925]                              │  nom_fons · año predicho   │
│        ╱     ╲                               │  Confianza: 82%            │
│   ●[img]──────●[img]──[shawl]                │  caption (catalán)         │
│                                              │                            │
│   ┌─ zoom ─┐  ⟳ reset  ⛶ fullscreen         │  ── 👗 Vestimenta ───────  │
│                                              │  [shawl][tuxedo][gloves]   │
│                                              │                            │
│                                              │  ── 📅 Año 1925 ─────────  │
│                                              │  [t][t][t][t][t][t]        │
│                                              │                            │
│                                              │  ── 📝 Palabras HTR ─────  │
│                                              │  etapa · vida · retrat     │
└──────────────────────────────────────────────┴────────────────────────────┘
  ~75% del ancho                                ~25% del ancho
```

### Móvil (≤768px)
El panel se colapsa. Un botón flotante `ⓘ` lo abre como drawer desde la derecha. El grafo ocupa toda la pantalla.

### Estado inicial
Sin estado vacío: al cargar, se pre-selecciona automáticamente el nodo `imagen` con mayor `degree_static`.

---

## Stack técnico

```
next@15            App Router, SSG, API routes
sigma@3            Motor de renderizado WebGL del grafo
graphology         Estructura de datos del grafo (compatible con sigma)
@react-sigma/core  Bindings React para sigma + graphology
@sigma/node-image  Renderer WebGL oficial para thumbnails circulares en nodos
graphology-gexf    Parser GEXF → graphology (resuelve atributos numéricos solo)
graphology-layout-forceatlas2  Layout pre-calculado en build
zustand            Estado global (selectedNodeId, activeLayers, hoveredNodeId)
framer-motion      Animación crossfade del panel lateral
tailwindcss@4      Estilos utility-first, dark mode
```

### Instalación
```bash
npx create-next-app@latest arenas-explorer --typescript --tailwind --app
cd arenas-explorer
npm install sigma graphology @react-sigma/core @sigma/node-image \
  graphology-gexf graphology-layout-forceatlas2 \
  zustand framer-motion
```

---

## Procesado de datos (build-time)

El GEXF se convierte a **dos ficheros JSON estáticos** durante el build de Next.js. El cliente **nunca** parsea el GEXF.

```
scripts/parse-gexf.ts
  ├── Entrada: grafo_multidimensional.gexf  +  input.csv
  ├── Parsea el GEXF con graphology-gexf
  ├── Cruza nodos imagen con las filas del CSV por filename=id
  ├── Ejecuta ForceAtlas2 (500 iteraciones) → coordenadas x,y
  ├── Escribe: public/data/graph.json
  └── Escribe: public/data/images-index.json
```

### `public/data/graph.json` — esquema exacto

```typescript
// Tipos TypeScript
type Dimension = "imagen" | "transcripcion" | "año" | "vestimenta";
type Relation  = "lleva_puesto" | "mismo_año" | "pertenece_a_año" | "contiene_palabra";

interface GraphNode {
  id:        string;      // filename.jpg | year_node_XXXX | label (prenda/palabra)
  label:     string;      // texto visible
  dimension: Dimension;
  community: number;      // 0–11 (Louvain)
  degree:    number;      // degree_static del GEXF
  x:         number;      // ForceAtlas2
  y:         number;      // ForceAtlas2
  // solo imagen:
  year?:      number;
  confianza?: number;     // 0–100
  // solo año:
  n_fotos?:   number;
}

interface GraphEdge {
  source:   string;       // id del nodo origen
  target:   string;       // id del nodo destino
  relation: Relation;
}

interface GraphData {
  nodes: GraphNode[];
  edges: GraphEdge[];
}
```

Ejemplo real de cada tipo de nodo:

```json
{ "id": "IAAH_GUDIOL_34303.jpg", "label": "IAAH_GUDIOL_34303.jpg",
  "dimension": "imagen", "community": 8, "degree": 5, "year": 1925,
  "confianza": 82.33, "x": 120.4, "y": -88.1 }

{ "id": "year_node_1925", "label": "1925", "dimension": "año",
  "community": 8, "degree": 367, "n_fotos": 4, "x": 130.0, "y": -90.0 }

{ "id": "shawl", "label": "shawl", "dimension": "vestimenta",
  "community": 10, "degree": 163, "x": 95.0, "y": -70.0 }

{ "id": "etapa", "label": "etapa", "dimension": "transcripcion",
  "community": 8, "degree": 1, "x": 115.0, "y": -82.0 }
```

Ejemplo de aristas:

```json
{ "source": "IAAH_GUDIOL_34303.jpg", "target": "year_node_1925",  "relation": "pertenece_a_año" }
{ "source": "IAAH_GUDIOL_34303.jpg", "target": "shawl",           "relation": "lleva_puesto" }
{ "source": "IAAH_GUDIOL_34303.jpg", "target": "etapa",           "relation": "contiene_palabra" }
{ "source": "IAAH_GUDIOL_34303.jpg", "target": "ACCB_681862.jpg", "relation": "mismo_año" }
```

### `public/data/images-index.json` — esquema exacto

```typescript
interface ImageMeta {
  caption:    string;   // descripción en catalán
  url:        string;   // URL pública directa a la imagen
  year_csv:   number;   // año del CSV (puede diferir del predicho)
  nom_fons:   string;   // fondo documental de origen
  nom_arxiu:  string;   // archivo histórico
  toponims:   string;   // lugar geográfico (puede ser "")
  noms_propis: string;  // personas (puede ser "")
}

type ImagesIndex = Record<string, ImageMeta>;
// clave = filename = id del nodo imagen en graph.json
```

Ejemplo:

```json
{
  "ACCB_681862_310012230001149.jpg": {
    "caption": "Retrat d'un nen no identificat a les escales de l'església de Santa Maria",
    "url": "https://static.arxiusenlinia.cultura.gencat.cat/active/GIAC/NOTEXTUAL/310/122/1/1149/ACCB_681862_310012230001149.jpg",
    "year_csv": 1950,
    "nom_fons": "Anton Moix",
    "nom_arxiu": "Arxiu Comarcal Conca de Barberà",
    "toponims": "Montblanc",
    "noms_propis": ""
  }
}
```

> Las URLs son directas y públicas. No se copian imágenes al proyecto. Ver sección "Notas críticas" para CORS.

---

## Diseño visual de nodos

### Paleta por dimensión

| `dimension`     | Color hex | Forma Sigma                    | Tamaño base  |
| --------------- | --------- | ------------------------------ | ------------ |
| `imagen`        | `#4A90D9` | `NodeProgramImage` (thumbnail) | degree-based |
| `año`           | `#E8604B` | `NodeProgramSquare`            | grande fijo  |
| `vestimenta`    | `#9B59B6` | `NodeProgramCircle`            | mediano      |
| `transcripcion` | `#27AE60` | `NodeProgramCircle`            | pequeño      |

### Tamaño de nodo
```typescript
// Proporcional al grado. Mínimo 6, máximo 30.
const nodeSize = (degree: number) => 6 + Math.log1p(degree) * 4;
```

### Colores de arista

| `relation`         | Color (por defecto) | Opacidad por defecto |
| ------------------ | ------------------- | -------------------- |
| `lleva_puesto`     | `#9B59B6`           | 0.6                  |
| `pertenece_a_año`  | `#E8604B`           | 0.6                  |
| `contiene_palabra` | `#27AE60`           | 0.6                  |
| `mismo_año`        | `#999999`           | **0** (oculta)       |

### Registro de programas en Sigma
```typescript
// En GraphCanvas.tsx → settings de SigmaContainer
import { NodeProgramImage } from "@sigma/node-image";
import NodeProgramCircle from "sigma/rendering/programs/node.circle";

const settings = {
  nodeProgramClasses: {
    image:   NodeProgramImage,
    circle:  NodeProgramCircle,
    square:  NodeProgramCircle, // usar hasta tener NodeProgramSquare disponible
  },
  // Asignar por dimensión en nodeReducer:
  // node.type = node.dimension === "imagen" ? "image" : "circle"
};
```

---

## Estructura de componentes y tipos

### Árbol de ficheros
```
src/
├── app/
│   ├── layout.tsx                    ← font Inter, metadata
│   └── page.tsx                      ← Shell split: grafo 75% + panel 25%
├── components/
│   ├── graph/
│   │   ├── GraphCanvas.tsx           ← SigmaContainer; carga graph.json; posiciones FA2
│   │   ├── GraphControls.tsx         ← botones zoom in/out, reset, pantalla completa
│   │   ├── LayerFilter.tsx           ← toggles por dimension + toggle mismo_año
│   │   └── GraphEvents.tsx           ← useRegisterEvents: clickNode, enterNode, leaveNode
│   └── panel/
│       ├── DetailPanel.tsx           ← contenedor derecho; scroll interno; AnimatePresence
│       ├── PhotoPreview.tsx          ← <img src={url}> + caption + año + confianza
│       ├── ConnectionSection.tsx     ← sección reutilizable con título + children
│       ├── ClothingChip.tsx          ← chip clickable de prenda (vestimenta)
│       ├── WordChip.tsx              ← chip clickable de palabra HTR
│       └── RelatedThumb.tsx          ← thumbnail 64px clickable de foto del mismo año
├── hooks/
│   ├── useGraphData.ts               ← fetch /data/graph.json + /data/images-index.json
│   ├── useNodeNeighbors.ts           ← retorna vecinos de un nodeId agrupados por relation
│   └── useLayerFilter.ts             ← devuelve nodeReducer + edgeReducer según activeLayers
├── store/
│   └── graphStore.ts                 ← Zustand store
└── types/
    └── graph.ts                      ← tipos GraphNode, GraphEdge, GraphData, ImageMeta
```

### `src/types/graph.ts`
```typescript
export type Dimension = "imagen" | "transcripcion" | "año" | "vestimenta";
export type Relation  = "lleva_puesto" | "mismo_año" | "pertenece_a_año" | "contiene_palabra";

export interface GraphNode {
  id:          string;
  label:       string;
  dimension:   Dimension;
  community:   number;
  degree:      number;
  x:           number;
  y:           number;
  year?:       number;
  confianza?:  number;
  n_fotos?:    number;
}

export interface GraphEdge {
  source:   string;
  target:   string;
  relation: Relation;
}

export interface GraphData {
  nodes: GraphNode[];
  edges: GraphEdge[];
}

export interface ImageMeta {
  caption:     string;
  url:         string;
  year_csv:    number;
  nom_fons:    string;
  nom_arxiu:   string;
  toponims:    string;
  noms_propis: string;
}

export type ImagesIndex = Record<string, ImageMeta>;
```

### `src/store/graphStore.ts`
```typescript
import { create } from "zustand";
import { Dimension } from "@/types/graph";

interface GraphStore {
  selectedNodeId:  string | null;
  hoveredNodeId:   string | null;
  activeLayers:    Set<Dimension>;
  showSameYear:    boolean;           // toggle para aristas mismo_año
  setSelected:     (id: string | null) => void;
  setHovered:      (id: string | null) => void;
  toggleLayer:     (d: Dimension) => void;
  toggleSameYear:  () => void;
}

export const useGraphStore = create<GraphStore>((set) => ({
  selectedNodeId: null,
  hoveredNodeId:  null,
  activeLayers:   new Set(["imagen", "transcripcion", "año", "vestimenta"]),
  showSameYear:   false,
  setSelected:    (id) => set({ selectedNodeId: id }),
  setHovered:     (id) => set({ hoveredNodeId: id }),
  toggleLayer:    (d) => set((s) => {
    const next = new Set(s.activeLayers);
    next.has(d) ? next.delete(d) : next.add(d);
    return { activeLayers: next };
  }),
  toggleSameYear: () => set((s) => ({ showSameYear: !s.showSameYear })),
}));
```

---

## Interacciones y comportamiento

### Flujo completo
1. App carga → fetch `graph.json` + `images-index.json` → Sigma renderiza con posiciones FA2
2. Se pre-selecciona el nodo `imagen` con mayor `degree` → panel muestra su detalle
3. Usuario hace pan/zoom con ratón/trackpad libremente en el grafo
4. **Click en nodo imagen** → `setSelected(id)` → panel hace crossfade con `AnimatePresence`
5. **Click en elemento del panel** (chip o thumbnail) → `setSelected(id)` → `sigma.getCamera().animate({...})`
6. **Click en nodo año/vestimenta/palabra** → panel muestra lista de fotos relacionadas

### Comportamiento del panel por tipo de nodo seleccionado

| Nodo seleccionado         | Contenido del panel                                                                                                                             |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `imagen`                  | Foto ampliada · caption · año predicho + confianza · sección Vestimenta (chips) · sección Mismo Año (thumbnails) · sección Palabras HTR (chips) |
| `año` (`year_node_1925`)  | Título "Año 1925 · N fotos" · grid de thumbnails de todas las fotos con `pertenece_a_año`                                                       |
| `vestimenta` (`shawl`)    | Título "Prenda: shawl · N fotos" · grid de thumbnails de fotos con `lleva_puesto`                                                               |
| `transcripcion` (`etapa`) | Título "Palabra: etapa · N fotos" · grid de thumbnails de fotos con `contiene_palabra`                                                          |

### Filtrado de capas (toggles en la toolbar)

```
[● imagen] [● año] [● vestimenta] [● palabras] | [○ mostrar mismo año]
```

- Toggles inactivos → `nodeReducer`: `hidden: true` para nodos de esa dimensión
- `edgeReducer`: oculta aristas cuyo `source` o `target` sea un nodo oculto
- `mismo_año` siempre oculto salvo toggle explícito (demasiado ruido: 1596 aristas)
- **Nunca eliminar nodos del grafo** — rompe el layout. Solo `hidden` en el reducer.

### Hover (ego-graph highlight)
```typescript
// nodeReducer durante hover
if (hoveredNodeId) {
  const neighbors = graph.neighbors(hoveredNodeId);
  const isRelevant = nodeId === hoveredNodeId || neighbors.includes(nodeId);
  return { ...node, color: isRelevant ? node.color : "#e0e0e0", zIndex: isRelevant ? 1 : 0 };
}
```

### Navegación de cámara al seleccionar nodo desde panel
```typescript
sigma.getCamera().animate(
  { x: node.x, y: node.y, ratio: 0.15 },
  { duration: 500, easing: "quadraticInOut" }
);
```

---

## Fases de implementación

### Fase 1 — Setup + datos *(1 día)*
- [ ] `npx create-next-app@latest arenas-explorer --typescript --tailwind --app`
- [ ] Instalar dependencias (ver sección Stack)
- [ ] Escribir `scripts/parse-gexf.ts`: lee el GEXF con `graphology-gexf`, cruza con CSV, ejecuta ForceAtlas2, escribe `public/data/graph.json` y `public/data/images-index.json`
- [ ] Añadir `"prebuild": "npx ts-node scripts/parse-gexf.ts"` en `package.json`
- [ ] Verificar manualmente que varias URLs del CSV devuelven imagen (abrir en navegador)

### Fase 2 — Grafo base *(2 días)*
- [ ] `GraphCanvas`: `SigmaContainer` con `graph` Graphology cargado desde `graph.json`; posiciones FA2 ya en el JSON, sin re-layout en cliente
- [ ] `nodeReducer` inicial: asignar color + tamaño (`nodeSize(degree)`) + `type` por `dimension`
- [ ] `edgeReducer` inicial: ocultar `mismo_año` por defecto
- [ ] `GraphEvents`: `clickNode` → `setSelected`; `enterNode` → `setHovered`; `leaveNode` → `setHovered(null)`
- [ ] `LayerFilter`: 4 toggles de dimensión + toggle "mismo año"
- [ ] `GraphControls`: zoom in/out (`sigma.getCamera().animatedZoom`), reset, fullscreen

### Fase 3 — Thumbnails en nodos *(1-2 días)*
- [ ] Registrar `NodeProgramImage` en `settings.nodeProgramClasses` de Sigma
- [ ] Pre-cargar texturas: antes de montar `SigmaContainer`, `Promise.all` sobre todos los nodos `imagen` → `new Image()` con la URL de `images-index.json`
- [ ] Añadir `image: url` al nodo en el grafo Graphology durante la precarga
- [ ] Fallback: si URL falla (CORS o 404), no asignar `image` → nodo renderiza con color sólido
- [ ] Nodos `año` con `type: "square"` (registrar `NodeProgramSquare` si disponible)

### Fase 4 — Panel lateral *(2 días)*
- [ ] `DetailPanel`: div fijo ancho 25%, height 100vh, scroll interno; `AnimatePresence` de Framer Motion en el contenido
- [ ] `PhotoPreview`: `<img src={meta.url}>` con `loading="lazy"` + caption + año predicho + confianza en %
- [ ] `useNodeNeighbors(id)`: extrae aristas del grafo Graphology, agrupa `source/target` por `relation`, devuelve `{ lleva_puesto: string[], pertenece_a_año: string[], contiene_palabra: string[] }`
- [ ] Secciones: `ConnectionSection` con título · `ClothingChip` · `WordChip` · `RelatedThumb` (64px)
- [ ] Click en chip/thumbnail → `setSelected(id)` → `sigma.getCamera().animate()`

### Fase 5 — Pulido *(1 día)*
- [ ] Hover: `nodeReducer` dinámico con ego-graph highlight (vecinos al 100%, resto al 15% opacidad)
- [ ] Tooltip flotante con `label` al hacer hover (div absoluto sobre el canvas)
- [ ] Responsive: `≤768px` → panel como drawer con botón `ⓘ` flotante; `useMediaQuery` o CSS container queries
- [ ] Loading skeleton mientras `graph.json` se carga (spinner centrado sobre el canvas)
- [ ] `<title>` y `<meta>` correctos en `layout.tsx`

---

## Notas de implementación críticas

### 1. GEXF namespace
El fichero usa `xmlns="http://www.gexf.net/1.2draft"` (no 1.3). `graphology-gexf` lo maneja. Si se parsea manualmente con `DOMParser`, usar el namespace correcto.

### 2. Atributos numéricos en el GEXF
Los `<attvalue for="N">` usan IDs numéricos. `graphology-gexf` los resuelve solo. Mapa de referencia por si se parsea a mano:
```
Nodos:  0=title  1=node_type  2=pk  3=group  4=size  5=color  6=dimension
        7=year   8=confianza  9=type  10=community  11=degree_static  12=n_fotos
Aristas: 13=relation  14=dimension
```

### 3. CORS de imágenes
Las fotos están en `static.arxiusenlinia.cultura.gencat.cat`. Para `<img>` en React funciona bien. Para **texturas WebGL en Sigma** el navegador puede bloquear por CORS. Solución:

```typescript
// app/api/img/route.ts — proxy de imagen en Next.js
export async function GET(req: Request) {
  const url = new URL(req.url).searchParams.get("url")!;
  const res = await fetch(url);
  return new Response(res.body, {
    headers: { "Content-Type": res.headers.get("Content-Type") ?? "image/jpeg" },
  });
}
// Uso: <img src={`/api/img?url=${encodeURIComponent(meta.url)}`} />
// O como textura Sigma: graph.setNodeAttribute(id, "image", `/api/img?url=...`)
```

### 4. Thumbnails en Sigma — orden de operaciones
```typescript
// CORRECTO: cargar imágenes ANTES de montar SigmaContainer
const [ready, setReady] = useState(false);

useEffect(() => {
  Promise.all(
    imageNodes.map(node =>
      new Promise<void>(resolve => {
        const img = new Image();
        img.crossOrigin = "anonymous";
        img.src = `/api/img?url=${encodeURIComponent(imagesIndex[node.id].url)}`;
        img.onload = () => {
          graph.setNodeAttribute(node.id, "image", img.src);
          graph.setNodeAttribute(node.id, "type", "image");
          resolve();
        };
        img.onerror = () => resolve(); // fallback: nodo sin imagen
      })
    )
  ).then(() => setReady(true));
}, []);

// Solo montar Sigma cuando ready === true
```

### 5. Aristas `mismo_año` y layout
Las 1 596 aristas foto↔foto son **necesarias** para que ForceAtlas2 agrupe las fotos por época en el layout. Incluirlas en el JSON pero ocultarlas en cliente:
```typescript
// edgeReducer
(edgeId, edge, sourceNode, targetNode, source, target) => {
  if (edge.relation === "mismo_año" && !showSameYear) return { ...edge, hidden: true };
  // ...
}
```

### 6. Posiciones fijas — sin re-layout en cliente
ForceAtlas2 se ejecuta **una sola vez** en `scripts/parse-gexf.ts` (500 iteraciones). Las `x,y` se guardan en `graph.json`. El cliente **no** ejecuta ningún layout. Al montar Sigma, los nodos aparecen directamente en sus posiciones finales.

### 7. Identificadores
- Nodo imagen: `id === filename` del CSV (ej: `"IAAH_GUDIOL_34303.jpg"`)
- Nodo año: `id === "year_node_" + year` (ej: `"year_node_1925"`)
- Nodo prenda: `id === label en inglés` (ej: `"shawl"`, `"tuxedo"`)
- Nodo palabra: `id === palabra transcrita` (ej: `"etapa"`, `"vida"`)

### 8. Nunca mutar el grafo para filtrar
```typescript
// MAL — rompe el layout FA2
graph.dropNode(nodeId);

// BIEN — ocultar sin mutar
nodeReducer: (nodeId, node) => ({
  ...node,
  hidden: !activeLayers.has(node.dimension),
})
```

