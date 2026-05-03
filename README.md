# TFM-Sara — Grafo de Conocimiento Multidimensional de Fotografías Históricas

Pipeline para analizar fotografías históricas sin etiquetar y organizarlas en un **grafo de conocimiento multidimensional**, donde cada dimensión representa una capa de información extraída automáticamente (transcripción de texto manuscrito, año estimado, y en el futuro: vestimenta, postura, etc.).

---

## Flujo principal

```
input/          →  pipeline/01  →  pipeline/02  →  pipeline/03  →  output/grafo_final/
(tus fotos)        Transcripción    Predicción año    Fusión           grafo_multidimensional.gexf
```

**El proceso completo se ejecuta abriendo en orden los 3 notebooks de `pipeline/`.**

---

## Estructura del proyecto

```
TFM-Sara/
│
├── pipeline/                          ← EJECUTAR EN ORDEN
│   ├── 01_transcripcion.ipynb        # Paso 1: OCR + HTR → grafo de palabras
│   ├── 02_prediccion_año.ipynb       # Paso 2: CLIP embeddings → predicción de año
│   └── 03_fusion_grafos.ipynb        # Paso 3: Fusión → grafo multidimensional
│
├── helpers/                           ← HERRAMIENTAS DE SOPORTE (ejecutar una sola vez)
│   └── generar_cerebro_años.ipynb    # Construye el índice vectorial Annoy
│
├── assets/                            ← MODELOS Y RECURSOS ESTÁTICOS
│   ├── handwritten_expert.pt         # Modelo HTR (handwritten text recognition)
│   ├── oda_giga_tokenizer.json       # Tokenizador a nivel de carácter
│   └── transcription-examples.txt   # Diccionario de corrección (dominio catalán)
│
├── support_data/                      ← CORPUS DE SOPORTE CON AÑO CONOCIDO
│   ├── input.csv                     # 312 fotografías etiquetadas con metadatos
│   ├── images/                       # Imágenes de referencia (con fecha conocida)
│   └── index/                        # Índice vectorial generado por helpers/
│       ├── support_database.ann      # Índice Annoy (embeddings CLIP)
│       └── metadata_index.json       # Mapping vector → metadatos
│
├── input/                             ← PON AQUÍ TUS FOTOS A ANALIZAR
│   └── (tus fotografías .jpg / .png)
│
└── output/                            ← RESULTADOS GENERADOS AUTOMÁTICAMENTE
    ├── transcripciones/               # Salida del Paso 1
    │   ├── transcripciones.csv
    │   ├── transcripciones.json
    │   └── knowledge_graph_palabras.gexf
    ├── predicciones_año/              # Salida del Paso 2
    │   ├── grafo_predicciones.json
    │   └── grafo_años.gexf
    └── grafo_final/                   # Salida del Paso 3 (resultado final)
        └── grafo_multidimensional.gexf
```

---

## Instrucciones de uso

### Requisitos previos

- Cuenta de Google con acceso a Google Drive y Google Colab (GPU T4)
- Carpeta `TFM-Sara/` subida a Google Drive con la estructura completa
- El modelo `assets/handwritten_expert.pt` debe estar presente

### Cómo analizar fotografías nuevas

1. **Copia tus fotografías** en la carpeta `input/`

2. **Abre en Colab** `pipeline/01_transcripcion.ipynb`
   → OCR con EasyOCR + reconocimiento HTR con el modelo experto
   → Corrige palabras con spell-checker con caché (rápido para palabras repetidas)
   → Genera `output/transcripciones/knowledge_graph_palabras.gexf`

3. **Abre en Colab** `pipeline/02_prediccion_año.ipynb`
   → Calcula embeddings visuales en batches de 32 con OpenCLIP (ViT-B-32) — ~20× más rápido
   → Busca los 5 vecinos más cercanos en el índice Annoy
   → Predice el año y la confianza por foto; conecta fotos del mismo año entre sí
   → Genera `output/predicciones_año/grafo_años.gexf`

4. **Abre en Colab** `pipeline/03_fusion_grafos.ipynb`
   → Fusiona todos los grafos de dimensiones activas
   → Normaliza atributos de nodos imagen tras la fusión
   → Genera `output/grafo_final/grafo_multidimensional.gexf`

5. **Visualiza** el fichero `.gexf` en [Gephi](https://gephi.org/) — ver sección Gephi más abajo

### Si el índice Annoy no existe (primera vez o corpus nuevo)

Ejecuta `helpers/generar_cerebro_años.ipynb` antes del Paso 2.
Lee `support_data/input.csv` y las imágenes en `support_data/images/`.
Procesa en batches de 32 (GPU T4) y construye el índice con todos los cores (`n_jobs=-1`).
Genera `support_data/index/support_database.ann` y `metadata_index.json`.

---

## Casos de uso frecuentes

### Caso 1 — Primera vez: quiero analizar mis fotos desde cero

> Situación: tienes fotos en `input/` y el índice Annoy no existe todavía.

```
Paso 0 (solo la primera vez):
  helpers/generar_cerebro_años.ipynb
  → Construye el índice vectorial del corpus de referencia
  → Resultado: support_data/index/support_database.ann + metadata_index.json

Paso 1: pipeline/01_transcripcion.ipynb
Paso 2: pipeline/02_prediccion_año.ipynb
Paso 3: pipeline/03_fusion_grafos.ipynb
```

---

### Caso 2 — Tengo los GEXFs intermedios y solo quiero generar el grafo final

> Situación: ya ejecutaste los pasos 1 y 2 y tienes:
> - `output/transcripciones/knowledge_graph_palabras.gexf` ✅
> - `output/predicciones_año/grafo_años.gexf` ✅
>
> Solo falta fusionarlos.

```
Ejecuta únicamente: pipeline/03_fusion_grafos.ipynb
→ Resultado: output/grafo_final/grafo_multidimensional.gexf
```

No es necesario volver a ejecutar los pasos 1 ni 2.

---

### Caso 3 — Añado fotos nuevas a las que ya tenía

> Situación: ya tienes un grafo generado y quieres incorporar fotografías adicionales.

```
1. Copia las fotos nuevas en input/  (pueden coexistir con las anteriores)
2. Ejecuta pipeline/01_transcripcion.ipynb  → sobrescribe knowledge_graph_palabras.gexf
3. Ejecuta pipeline/02_prediccion_año.ipynb → sobrescribe grafo_años.gexf
4. Ejecuta pipeline/03_fusion_grafos.ipynb  → regenera grafo_multidimensional.gexf
```

⚠️ Los pasos 1 y 2 procesan **todas** las fotos que haya en `input/` en cada ejecución.
Si quieres mantener el resultado anterior, haz una copia de seguridad de `output/` antes.

---

### Caso 4 — Solo quiero regenerar las transcripciones (cambié los parámetros OCR)

> Situación: modificaste el umbral de confianza o las stop words en el paso 1 y quieres rehacerlo sin tocar las predicciones de año.

```
1. Ejecuta pipeline/01_transcripcion.ipynb  → sobrescribe knowledge_graph_palabras.gexf
2. Ejecuta pipeline/03_fusion_grafos.ipynb  → regenera grafo_multidimensional.gexf
   (el grafo_años.gexf del paso 2 se reutiliza tal cual)
```

---

### Caso 5 — Solo quiero regenerar las predicciones de año (cambié el índice Annoy)

> Situación: actualizaste el corpus de soporte y regeneraste el índice.

```
1. helpers/generar_cerebro_años.ipynb       → regenera support_database.ann
2. pipeline/02_prediccion_año.ipynb         → sobrescribe grafo_años.gexf
3. pipeline/03_fusion_grafos.ipynb          → regenera grafo_multidimensional.gexf
   (el knowledge_graph_palabras.gexf del paso 1 se reutiliza tal cual)
```

---

### Caso 6 — El grafo en Gephi se ve caótico, no veo constelaciones

> Situación: abriste el GEXF en Gephi pero los nodos están dispersos sin estructura visible.

```
1. Layout → Force Atlas 2
   · Scaling: 10
   · Gravity: 1.0
   · LinLog mode: activado
   · Ejecuta hasta que converja (~30 segundos)

2. Appearance → Nodes → Partition → dimension
   · Asigna colores: imagen=#4A90D9, transcripcion=#5CB85C, año=#1A3A5C

3. Si hay demasiado ruido: Filters → Attributes → Equal → dimension = "año"
   → Muestra solo fotos + hubs de año → las constelaciones son inmediatamente visibles

4. Vuelve a activar todas las dimensiones para el grafo completo
```

Las constelaciones son los grupos de fotos conectadas por aristas `mismo_año`.
Los hubs oscuros grandes (`#1A3A5C`) son los centros de cada constelación.

---

### Caso 7 — Quiero añadir una nueva dimensión (postura, vestimenta…)

> Situación: tienes un modelo que clasifica poses o ropa y quieres integrarlo.

```
1. Crea pipeline/04_postura.ipynb:
   · Procesa las imágenes de input/
   · Para cada imagen, crea un nodo con dimension='postura', color='#E67E22', group=4
   · Crea aristas relation='tiene_postura', dimension='postura'
   · Exporta a output/postura/grafo_postura.gexf

2. En pipeline/03_fusion_grafos.ipynb, descomenta esta línea del registro:
   # '/content/drive/MyDrive/TFM-Sara/output/postura/grafo_postura.gexf'

3. Ejecuta pipeline/03_fusion_grafos.ipynb → nueva dimensión integrada automáticamente
```

---

## Schema del grafo multidimensional

### Nodos

| Tipo | `dimension` | `color` | `group` | `size` | Descripción |
|------|-------------|---------|---------|--------|-------------|
| Imagen | `imagen` | `#4A90D9` azul | 1 | 25 | Foto a analizar |
| Palabra | `transcripcion` | `#5CB85C` verde | 2 | 10–40 ★ | Palabra transcrita del texto manuscrito |
| Hub-Año | `año` | `#1A3A5C` azul oscuro | 3 | 20–60 ★★ | Nodo central por década/año predicho |
| *(futuro)* Postura | `postura` | `#E67E22` naranja | 4 | — | Etiqueta de pose corporal detectada |
| *(futuro)* Vestimenta | `vestimenta` | `#9B59B6` morado | 5 | — | Categoría de ropa clasificada |

★ El tamaño de las palabras escala proporcionalmente a su frecuencia: palabras usadas en más fotos aparecen más grandes.
★★ El tamaño de los hubs de año escala con el número de fotos asignadas a ese año.

### Aristas

| `relation` | `dimension` | `weight` | Descripción |
|-----------|-------------|----------|-------------|
| `contiene_palabra` | `transcripcion` | 1.0 | Imagen → Palabra transcrita |
| `pertenece_a_año` | `año` | 1.5 | Imagen → Hub del año predicho |
| `mismo_año` | `año` | 0.8 | Imagen ↔ Imagen (≤5 vecinos, misma época) |
| *(futuro)* `tiene_postura` | `postura` | 1.0 | Imagen → Nodo de postura |
| *(futuro)* `tiene_vestimenta` | `vestimenta` | 1.0 | Imagen → Nodo de vestimenta |

El atributo `dimension` en nodos y aristas permite **filtrar capas individualmente en Gephi** (`Filters → Attributes → Equal → dimension = "transcripcion"`).

---

## Herramientas de visualización

El GEXF generado por el Paso 3 incluye el **namespace `viz:`** (estándar GEXF 1.2) que embebe colores, tamaños y formas directamente en el fichero. Cualquier herramienta compatible lo abre ya estilizado sin configuración manual.

### Comparativa de opciones

| Herramienta | Tipo | Instalar | GEXF nativo | Escala | Mejor para |
|-------------|------|----------|-------------|--------|------------|
| **Gephi** | Desktop | Sí (Java) | ✅ | ~100k nodos | Exploración / análisis / TFM |
| **Gephi Lite** | Web (browser) | No | ✅ | ~50k nodos | Compartir sin instalar nada |
| **Cosmograph** | Web (WebGL) | No | ❌ (CSV/JSON) | Millones | Datasets muy grandes |
| **pyvis HTML** | Web (ya generado) | No | — | ~5k nodos | Revisión rápida, incrustar |
| **Retina** | Web (browser) | No | ✅ | ~20k nodos | Alternativa ligera a Gephi Lite |

**Recomendación para este proyecto:**
- Exploración y capturas para el TFM → **Gephi desktop**
- Compartir resultados online → **Gephi Lite** (gephi.org/gephi-lite) — arrastra el `.gexf`
- Revisión rápida durante el desarrollo → el **HTML de pyvis** ya generado en `output/transcripciones/`

### Apariencia preconfigurada (viz: namespace)

El Paso 3 inyecta automáticamente en el GEXF:

```xml
<viz:color r="74" g="144" b="217" a="255"/>   <!-- color por dimensión -->
<viz:size value="25.0"/>                        <!-- tamaño escalado -->
<viz:shape value="disc"/>                       <!-- disc / square / diamond -->
```

Al abrir el fichero en Gephi o Gephi Lite, los nodos ya tienen el color y tamaño correctos. **No es necesario configurar nada.**

### Workflow en Gephi (solo para ajuste de layout)

La apariencia visual ya está aplicada. Solo queda distribuir los nodos en el espacio:

1. Abre `output/grafo_final/grafo_multidimensional.gexf`
   → Los colores y tamaños ya están aplicados al abrir

2. **Layout → Force Atlas 2** con estos parámetros:
   - `Scaling`: 10
   - `Gravity`: 1.0
   - `LinLog mode`: activado
   - Ejecuta ~30 segundos hasta que converja

3. **Si el grafo se ve saturado** — filtra para trabajar por capas:
   `Filters → Attributes → Equal → dimension = "año"`
   → Solo fotos y hubs de año → las constelaciones son inmediatamente visibles

4. **Etiquetar los hubs de año**: clic derecho en un hub → *Show label*, o activa labels globales y usa `Size` para que solo los nodos grandes muestren etiqueta

### Constelaciones esperadas

Con ~200 fotos el grafo mostrará clusters por época:
- Los hubs oscuros (`#1A3A5C`) son los centros de cada constelación — uno por año predicho
- Las fotos azules (`#4A90D9`) orbitan alrededor de su hub y se conectan entre sí por `mismo_año`
- Las palabras verdes (`#5CB85C`) más grandes son las más compartidas entre fotos de la misma época

---

## Añadir una nueva dimensión

1. Crea `pipeline/04_postura.ipynb` (o `04_vestimenta.ipynb`) que:
   - Procese las imágenes de `input/`
   - Añada `dimension='postura'`, `color='#E67E22'`, `group=4` a los nodos
   - Añada `dimension='postura'` a las aristas
   - Exporte a `output/postura/grafo_postura.gexf`

2. En `pipeline/03_fusion_grafos.ipynb`, descomenta la línea correspondiente en el registro de dimensiones (ya está preparada como comentario)

3. Re-ejecuta el Paso 3 — el grafo fusionado incluirá la nueva dimensión automáticamente

---

## Dimensiones del grafo

| Dimensión     | Estado    | Notebook             | Descripción                              |
|---------------|-----------|----------------------|------------------------------------------|
| Transcripción | ✅ Activo | `01_transcripcion`   | Texto manuscrito extraído por HTR        |
| Año           | ✅ Activo | `02_prediccion_año`  | Año estimado por similitud visual (CLIP) |
| Postura       | 🔜 Futuro | `04_postura`         | Detección de pose corporal               |
| Vestimenta    | 🔜 Futuro | `04_vestimenta`      | Clasificación de tipo de ropa            |

---

## Stack técnico

| Componente             | Tecnología                                         |
|------------------------|----------------------------------------------------|
| Detección de texto     | EasyOCR (CRAFT) — español + catalán               |
| Reconocimiento HTR     | PyTorch + ODAOCR (`handwritten_expert.pt`)         |
| Corrección ortográfica | pyspellchecker con caché por palabra               |
| Embeddings visuales    | OpenCLIP ViT-B-32 (laion2b) — inferencia en batch |
| Búsqueda por similitud | Annoy (distancia angular) — índice paralelo        |
| Construcción de grafos | NetworkX → GEXF                                    |
| Visualización          | Gephi (principal) / pyvis (HTML interactivo)       |
| Entorno de ejecución   | Google Colab (GPU T4) + Google Drive               |

---

## Fuente de datos de soporte

312 fotografías históricas de archivos catalanes (Arxiu Nacional de Catalunya, arxius comarcals)
con metadatos: fondo, descripción, topónimos, nombres propios y año conocido.
Fuente: [arxiusenlinia.cultura.gencat.cat](https://arxiusenlinia.cultura.gencat.cat)
