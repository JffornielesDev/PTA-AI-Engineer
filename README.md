# TomatoVLM · Identificación de problemas en hojas de tomate

Demostración en Google Colab que identifica enfermedades o plagas en una fotografía de una hoja de tomate y consulta documentos oficiales para mostrar información con citas.

Combina **fine-tuning parcial de CLIP** para las imágenes y **RAG** para consultar la documentación. Los resultados se presentan en español.

## Archivos necesarios

Para ejecutar la demo solo necesitas estos dos archivos del repositorio:

| Archivo | Para qué sirve |
| --- | --- |
| [TomatoVLM_Demo.ipynb](TomatoVLM_Demo.ipynb) | Cuaderno que se ejecuta en Google Colab. |
| [TomatoVLM_Demo_documentos.zip](TomatoVLM_Demo_documentos.zip) | Documentos y metadatos que se cargan en el paso 6. |

El dataset de imágenes y los tres modelos se descargan automáticamente durante la ejecución. No necesitas subir imágenes para entrenar ni descargar los modelos manualmente. Tampoco necesitas los scripts de desarrollo ni los cuadernos de versiones anteriores.

## Requisitos

- Una cuenta de Google para ejecutar el cuaderno en [Google Colab](https://colab.research.google.com/).
- Un entorno de Colab con **GPU NVIDIA**; para esta demo se propone **T4**.
- Conexión a Internet para descargar bibliotecas, modelos e imágenes.
- El ZIP de documentos descargado en tu ordenador.

No necesitas una cuenta de Hugging Face ni configurar `HF_TOKEN`: se utilizan modelos públicos. El cuaderno no monta Google Drive.

## Cómo ejecutar la demo

1. Descarga [TomatoVLM_Demo.ipynb](TomatoVLM_Demo.ipynb) y [TomatoVLM_Demo_documentos.zip](TomatoVLM_Demo_documentos.zip). En GitHub, abre cada archivo y utiliza la opción de descargar el archivo original. **Mantén el ZIP comprimido**.
2. Abre [Google Colab](https://colab.research.google.com/) y utiliza **Archivo → Subir cuaderno** para abrir el archivo `.ipynb`. También puedes abrirlo desde la pestaña GitHub del selector de cuadernos utilizando la dirección del repositorio.
3. Selecciona **Entorno de ejecución → Cambiar tipo de entorno de ejecución → GPU T4** y guarda el cambio.
4. Ejecuta las celdas en orden, desde el paso 1 hasta el 7. Si un paso contiene varias celdas, ejecútalas todas de arriba abajo.
5. Cuando el paso 6 lo solicite, selecciona **`TomatoVLM_Demo_documentos.zip`** de tu ordenador. No selecciones el ZIP completo del repositorio que genera el botón «Download ZIP» de GitHub.
6. En el paso 7, elige una fotografía de prueba o sube una foto propia y ejecuta la celda de análisis.

La primera ejecución necesita descargar los modelos y preparar el dataset. Espera a que termine cada celda antes de continuar.

| Paso | Qué hace | Qué puedes consultar |
| --- | --- | --- |
| 1 · Preparar | Instala las bibliotecas y comprueba la GPU. | Estado de la preparación. |
| 2 · Imágenes | Descarga y separa fotos para entrenamiento, validación y prueba. | Ejemplos y número de fotos. |
| 3 · CLIP original | Evalúa el modelo antes del ajuste. | Resultado inicial. |
| 4 · Entrenamiento | Ajusta la proyección visual de CLIP con las fotos de entrenamiento. | Progreso del aprendizaje. |
| 5 · Comparación | Evalúa el modelo ajustado sobre las mismas fotos de prueba. | Aciertos antes y después y detalle por clase. |
| 6 · Documentos | Lee los PDF, prepara la búsqueda y carga Qwen. | Cobertura documental por problema. |
| 7 · Consulta | Analiza una foto y consulta la documentación cuando procede. | Identificación, informe, extractos y fuentes. |

## Cómo repetir las pruebas

- **Otra fotografía:** repite la celda de consulta del paso 7; no necesitas entrenar otra vez.
- **Otro entrenamiento con la misma configuración:** repite los pasos 4 y 5. El entrenamiento vuelve a partir de la proyección original de CLIP.
- **Otros documentos:** activa `subir_otro_zip` en la celda de carga del paso 6, ejecútala y vuelve a ejecutar las celdas siguientes del paso 6 y la consulta del paso 7.
- **Sesión nueva o entorno eliminado:** vuelve a ejecutar todos los pasos y a cargar el ZIP.

Para una presentación, prepara los pasos 1–6 con antelación y mantén la sesión activa. Durante la demo basta con repetir la consulta del paso 7.

## Los tres modelos

| Modelo | Función | Entrenamiento en este proyecto |
| --- | --- | --- |
| [CLIP ViT-B/32](https://huggingface.co/openai/clip-vit-base-patch32) | Clasifica la imagen entre las diez clases de la demo. | **Fine-tuning supervisado parcial** de la proyección visual. |
| [Multilingual E5 Small](https://huggingface.co/intfloat/multilingual-e5-small) | Representa los textos numéricamente y permite buscar fragmentos por similitud de significado. | Se utiliza preentrenado, sin ajustar sus pesos. |
| [Qwen3-VL-2B-Instruct](https://huggingface.co/Qwen/Qwen3-VL-2B-Instruct) | Selecciona las evidencias recuperadas para los apartados del informe. | Se utiliza preentrenado, sin ajustar sus pesos. |

**Flujo:** fotografía → CLIP → búsqueda documental con E5 → selección de evidencias con Qwen → informe con citas.

### Fine-tuning de CLIP

Se ajustan los **393.216 pesos de la proyección visual preentrenada**. Los codificadores de imagen y texto permanecen congelados. Se usa una pérdida de clasificación de entropía cruzada y el optimizador AdamW; no se utiliza LoRA.

La configuración inicial reserva, por clase, 150 fotos para entrenar, 30 para validar y 50 para probar. Se realizan 30 épocas y se selecciona el ajuste con menor pérdida de validación, incluyendo el modelo original como candidato. Por tanto, si ningún ajuste mejora la validación, se conserva la proyección original.

El paso 5 muestra la mejora o el empeoramiento medido: el entrenamiento no garantiza una mayor precisión.

### Consulta documental mediante RAG

Los PDF se dividen en fragmentos asociados a su documento y página. E5 calcula sus representaciones numéricas, llamadas *embeddings*. Tras la identificación de CLIP, se buscan fragmentos relacionados con la descripción, los síntomas y las medidas de prevención o control. La búsqueda combina similitud semántica con nombres, variantes y contexto documental.

Qwen selecciona identificadores de evidencias. El código comprueba esa selección y construye el informe con **extractos literales y citas**. Si la salida de Qwen no es válida, se utiliza una selección automática de evidencias disponibles y se indica en el detalle de la respuesta.

Los PDF no se utilizan para hacer fine-tuning de Qwen. Cuando falta información, se indica; el informe no incorpora explicaciones libres procedentes del conocimiento previo del modelo.

## Documentos incluidos

El ZIP contiene:

- `guia_GIP_solanaceas.pdf`: guía de gestión integrada de plagas de solanáceas del MAPA, edición de 2021.
- `AMISTAR_TOP_25407.pdf`: ficha oficial de producto.
- `MILBEKNOCK_24062.pdf`: ficha oficial de producto.
- `sources.csv`: títulos, enlaces, nombres de archivo y huellas de comprobación de los documentos.
- `approved_evidence.csv`: extractos revisados; es un complemento y no limita toda la búsqueda.
- `PROCEDENCIA.md`: nota sobre el origen de los documentos.

No necesitas descomprimir, modificar ni subir estos archivos por separado: carga el ZIP completo en el paso 6.

Para utilizar documentos diferentes, prepara un ZIP con los documentos y un `sources.csv` siguiendo la estructura del incluido. La columna `filename` debe señalar los archivos que realmente contiene el ZIP. Se admiten subcarpetas. Si cambias un documento, actualiza sus metadatos y su huella; los rangos de páginas verificados de la guía original no se aplican automáticamente a otras versiones.

## Almacenamiento y conservación

La demo funciona con almacenamiento temporal de Colab y memoria de la sesión:

- El ajuste de CLIP se mantiene en memoria; **no se exporta un modelo entrenado**.
- Los PDF se extraen en `/content/tomato_demo/fuentes`.
- Los fragmentos y sus vectores se mantienen en memoria; no hay una base de datos persistente.
- No se guardan automáticamente métricas, informes ni un historial de consultas en Drive o GitHub.

Cuando Colab elimina el entorno se pierden esos datos de trabajo. Un reinicio de Python también elimina los modelos ajustados y el índice que estaban en memoria. **Guardar el cuaderno con sus salidas no guarda los pesos entrenados ni el índice del RAG.**

## Problemas habituales

| Situación | Qué hacer |
| --- | --- |
| El cuaderno indica que falta GPU. | Activa una GPU desde la configuración del entorno y vuelve al paso 1. |
| Tras instalar bibliotecas pide reiniciar. | Reinicia la sesión de Python desde el menú de Colab y ejecuta de nuevo desde el paso 1. |
| Aparece un aviso sobre `HF_TOKEN`. | El acceso a los modelos públicos no requiere ese secreto. Utiliza la versión actual del cuaderno, que configura acceso público explícito. |
| El ZIP da error o falta un documento. | Comprueba que has descargado el archivo original `TomatoVLM_Demo_documentos.zip` y que no has seleccionado el ZIP de todo el repositorio. Si es un paquete propio, revisa `sources.csv` y los nombres de archivo. |
| Identifica una plaga, pero falta información documental. | Revisa la cobertura del paso 6 y las fuentes desplegables. Acertar la imagen y disponer de documentos son comprobaciones independientes. |
| La identificación aparece «Por confirmar». | Prueba otra foto cercana, enfocada y bien iluminada. La consulta documental se omite cuando la identificación no supera el criterio de aceptación. |
| Se desconecta o elimina el entorno. | Reconecta y vuelve a ejecutar los pasos necesarios; si es un entorno nuevo, comienza desde el paso 1. |

## Alcance de la demostración

La clasificación utiliza imágenes de tomate del [repositorio PlantVillage empleado por el cuaderno](https://github.com/ai-agriculture-circuits-and-systems/plant_village), con una revisión fijada para reproducir la descarga. Las fotos del dataset se tomaron en condiciones controladas; los resultados pueden cambiar con imágenes de campo.

La guía incluida aporta cobertura para siete de las nueve clases no sanas de la demo. No se atribuye cobertura de esa guía a Septoria ni a mancha diana. Las puntuaciones individuales de CLIP son relativas entre las diez opciones y no representan una certeza diagnóstica.

El informe permite consultar lo que dicen las fuentes cargadas y su procedencia. La fecha y el ámbito de los documentos deben tenerse en cuenta al interpretar medidas de control. Esta demostración no sustituye una evaluación agronómica.
