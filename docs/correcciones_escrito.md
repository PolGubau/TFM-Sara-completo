# Discrepancias entre `docs/escrito.md` y la implementación real

Revisión línea a línea del texto frente al código de `pipeline/` y `helpers/`.

## 1. Formato del grafo: RDF vs GEXF (línea 12, 18)

**Texto:** "toda la información obtenida... se traduce al **estándar RDF** y se integra
unificadamente en el grafo de conocimiento" / "el fondo en un grafo de conocimiento o
**base de datos interconectada**".

**Código:** No hay ninguna dependencia de RDF (`rdflib`, `owlready2`, SPARQL, etc.) en
ningún notebook. Todo el pipeline construye el grafo con `networkx.Graph()` y lo exporta
con `nx.write_gexf(...)` (ver `02_prediccion_año.ipynb`, `03_deteccion_vestimenta.ipynb`,
`99_fusion_grafos.ipynb`). GEXF es un formato de intercambio para Gephi, no un estándar de
la web semántica (RDF/OWL). No existe ninguna capa de tripletas sujeto-predicado-objeto.

## 2. Reconocimiento de Entidades Nombradas — no existe (línea 18)

**Texto:** "el texto bruto extraído... se somete a procesos de **Reconocimiento de
Entidades Nombradas (NER)** para normalizar e identificar unívocamente a los personajes
históricos" / "cada **personaje documentado** se convierte en un nodo semántico".

**Código:** `01_transcripcion.ipynb` únicamente detecta cajas de palabras con EasyOCR
(`segment_words`) y transcribe cada recorte con el modelo HTR (`get_image_transcriptions`).
No hay ninguna librería de NER (spaCy, transformers NER, etc.) ni lógica de
desambiguación/entity linking. El resultado son palabras sueltas transcritas; en la fusión
(`99_fusion_grafos.ipynb`) estas palabras se añaden como nodos de dimensión
`transcripcion`, no como nodos de tipo "personaje" o "persona". No hay identificación
unívoca de individuos históricos en ningún punto del pipeline.

## 3. "Del texto al tiempo" — la fecha se predice por imagen, no por texto (línea 10, 12)

**Texto:** "Del **texto** al tiempo (Fase 2): Algoritmos de análisis **probabilístico y
RegEx** deducen y asignan fechas precisas a documentos que carecían de datación original."

**Código:** `02_prediccion_año.ipynb` no usa expresiones regulares ni analiza el texto
transcrito en absoluto. La fecha se predice por **similitud visual**: se generan embeddings
CLIP (`open_clip`, ViT-B-32) de cada foto nueva, se buscan los 5 vecinos más cercanos en un
índice Annoy construido sobre 312 fotos de referencia con año conocido
(`support_database.ann`, generado en `helpers/generar_cerebro_años.ipynb`), y el año
predicho es la media de los años de esos vecinos. No hay `import re`, ni ningún módulo de
NLP/regex en ese notebook. La fase debería describirse como "de la imagen al tiempo"
(image-to-time), no "del texto al tiempo".

## 4. Vestuario: transformer (OWLv2), no CNN (línea 11)

**Texto:** "De la imagen al concepto (Fase 3): **Redes neuronales convolucionales**
analizan visualmente las fotografías para categorizar el vestuario".

**Código:** `03_deteccion_vestimenta.ipynb` usa `Owlv2ForObjectDetection`
(`google/owlv2-base-patch16-ensemble`), un detector de objetos **zero-shot basado en
Vision Transformer** (ViT) con texto guía (`text_labels`), no una CNN clásica. Tampoco se
trata de una clasificación entrenada específicamente sobre vestuario histórico: es
detección abierta por similitud texto-imagen contra una lista fija de 34 etiquetas en
inglés (p. ej. "tailcoat", "flamenco dress", "top hat").

## 5. "Segmentar" imágenes — en realidad es detección de cajas (línea 17)

**Texto:** "se emplean modelos de Aprendizaje Profundo... para **segmentar** las imágenes
e identificar elementos visuales clave".

**Código:** OWLv2 (`post_process_grounded_object_detection`) devuelve **bounding boxes**
(`boxes`, `scores`, `text_labels`), no máscaras de segmentación a nivel de píxel. No hay
ningún modelo de segmentación semántica/instancia en el repositorio.

## 6. OCR + "análisis de maquetación" (línea 9)

**Texto:** "El reconocimiento óptico de caracteres (OCR) **y análisis de maquetación**
transforman el documento visual en **texto estructurado**".

**Código:** El pipeline usa EasyOCR únicamente para detección de palabras individuales
(`paragraph=False`, CRAFT) más un modelo HTR propio para transcribir cada recorte. No hay
análisis de maquetación/layout (columnas, párrafos, tablas, lectura en orden) ni ninguna
estructura jerárquica del texto: la salida final que llega al grafo son palabras sueltas
con su frecuencia (nodos verdes de tamaño ∝ frecuencia), no "texto estructurado".

## 7. Nodos "Actor/Actriz", "Espacio", "Estudio", "Producción" no existen (línea 4)

**Texto:** El ejemplo de grafo describe nodos de tipo Actor/Actriz, Producción/evento
teatral, Espacio (sala), Estudio (fotógrafo) y Vestuario, conectados con relaciones
`actúa en`, `representado en`, `estrenado en`, `retratada en`, `autoría de`,
`muestra vestuario`.

**Código:** Los tipos de nodo realmente generados por el pipeline (`99_fusion_grafos.ipynb`)
son únicamente: `imagen` (foto), `transcripcion` (palabra), `año` (hub temporal) y
`vestimenta` (prenda detectada), unidos por relaciones `pertenece_a_año`, `mismo_año` y
`lleva_puesto`. No existen nodos de persona/actor, sala/teatro, estudio fotográfico o
producción teatral, ni relaciones de autoría o representación. El ejemplo del apartado 4.1
es puramente ilustrativo/aspiracional y no corresponde a ninguna salida real del código.

## 8. Documentos manuscritos "coetáneos" separados de las fotos (línea 15, 17)

**Texto:** describe dos flujos paralelos de digitalización: fotografías por un lado y un
"conjunto de documentos manuscritos coetáneos" dedicados a describir la identidad de los
personajes, procesados por separado.

**Código:** `01_transcripcion.ipynb` procesa un único directorio
(`input/transcriptions`) con un solo bucle de OCR/HTR; no hay separación de código entre
"documentación en papel" y "soportes fotográficos", ni rutas/celdas distintas para cada
tipo de fuente. Todo el corpus de imágenes pasa por el mismo pipeline de segmentación de
palabras + transcripción.

## Resumen

| Afirmación del texto | Realidad en el código |
|---|---|
| Grafo en estándar RDF | GEXF vía NetworkX (`nx.write_gexf`) |
| NER para identificar personajes únicos | No existe; solo palabras transcritas sueltas |
| Fechas por RegEx/análisis probabilístico del texto | Fechas por similitud visual CLIP + Annoy sobre imágenes de referencia |
| CNN para vestuario | OWLv2 (Vision Transformer, zero-shot, detección por cajas) |
| Segmentación de imágenes | Detección de bounding boxes, sin máscaras |
| OCR + análisis de maquetación → texto estructurado | OCR de palabras sueltas (EasyOCR + HTR), sin estructura de layout |
| Nodos Actor, Espacio, Estudio, Producción | Solo nodos imagen, palabra, año y vestimenta |
| Dos flujos paralelos (fotos vs. manuscritos) | Un único pipeline sobre una misma carpeta de imágenes |
