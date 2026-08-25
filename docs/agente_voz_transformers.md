# Agente de voz local con Transformers

## 1. Arquitectura de referencia

La arquitectura que se probara en el TFM sera un pipeline modular de conversacion por turnos:

```text
Microfono
    |
    v
Whisper / faster-whisper (STT)
    |
    v
Modelo Transformer instruct (LLM)
    |
    v
F5-TTS o Fish Speech (TTS)
    |
    v
Altavoces
```

Un orquestador sencillo en Python conectara las etapas y conservara el historial de la conversacion. Cada componente recibira una entrada definida y devolvera una salida definida:

| Etapa | Entrada | Salida | Candidatos |
|---|---|---|---|
| STT | Audio WAV o bloque de audio | Texto de la intervencion | Whisper o `faster-whisper` |
| LLM | Historial y texto del usuario | Texto de respuesta | Qwen, Llama, Gemma o Mistral mediante `transformers` |
| TTS | Texto de respuesta | Audio WAV o array de audio | F5-TTS o Fish Speech |
| Orquestacion | Salidas y errores de cada etapa | Turno completo | Python |

Esta separacion permite cambiar el STT, el LLM o el TTS sin rediseñar toda la aplicacion. Tambien permite medir la latencia y la calidad de cada etapa por separado. La primera version sera por turnos y con archivos o buffers de audio; el streaming y las interrupciones se dejaran como una ampliacion posterior.

La arquitectura es open source en el sentido de que utiliza codigo y modelos publicados para su ejecucion local, pero las licencias deben comprobarse individualmente. En particular, los pesos de F5-TTS y Fish Speech tienen condiciones distintas de las de sus repositorios de codigo.

## 2. Objetivo

Este documento propone una segunda implementacion local del agente de voz utilizando directamente modelos de Hugging Face y la libreria `transformers`.

La implementacion se plantea como alternativa experimental a `Ollama_Local_Agent.ipynb`. El notebook comparara Transformers y Ollama dentro de la misma cadena de voz:

```text
Microfono -> STT -> LLM -> TTS -> Altavoces
```

La propuesta inicial es:

| Etapa | Componente propuesto | Funcion |
|---|---|---|
| STT | `faster-whisper` o Whisper | Convierte el audio en texto |
| LLM | Qwen instruct mediante `transformers` | Genera la respuesta |
| TTS | Piper, o un modelo compatible con Transformers | Convierte texto en audio |
| Captura y salida | `sounddevice` y `soundfile` | Graba y reproduce audio |
| Orquestacion | Python | Coordina los turnos |

El agente sigue siendo un pipeline STT-LLM-TTS. No es un modelo Speech-to-Speech nativo, porque el audio no entra directamente en el LLM ni sale directamente de el.

## 2. Por que usar Transformers en lugar de Ollama

No se puede afirmar de forma general que Transformers sea mejor que Ollama. Son capas diferentes:

- **Transformers** es una libreria de Python para cargar y ejecutar modelos. Permite controlar directamente el tokenizer, el dispositivo, la cuantizacion, el tipo numerico, la generacion y el flujo de datos.
- **Ollama** es un runtime y servidor local que empaqueta la ejecucion del modelo y la expone mediante una API. Simplifica la instalacion, la descarga, el cambio de modelos y el consumo desde otros programas.

Transformers puede ser mejor para este TFM cuando se quiera:

1. Medir por separado memoria, tiempo de carga y latencia de generacion.
2. Integrar el modelo en el mismo proceso que STT y TTS.
3. Experimentar con cuantizacion, `device_map`, `torch_dtype`, logits o criterios de parada.
4. Reproducir con precision la configuracion del modelo y del tokenizer.
5. Sustituir el modelo por otra arquitectura de Hugging Face sin depender de un servidor intermedio.
6. Estudiar el comportamiento interno del pipeline y documentar sus decisiones tecnicas.

Ollama sigue siendo preferible cuando se busca una prueba local sencilla, un proceso separado, una API estable o un consumo reducido de tiempo de desarrollo. Para comparar justamente ambas alternativas hay que mantener constantes el modelo, la cuantizacion, el prompt, el hardware y los parametros de generacion. Comparar `qwen2.5:3b` en Ollama con `Qwen2.5-7B-Instruct` en Transformers no seria una comparacion del runtime, sino tambien del tamaño del modelo.

### 2.1 Modelos candidatos

Todos estos modelos utilizan arquitecturas Transformer y pueden cargarse, cuando su arquitectura este soportada, mediante `transformers`. Las etiquetas como "mejor en español" o "mas natural" deben tratarse como hipotesis y comprobarse con el conjunto de pruebas del TFM.

| Modelo | Identificador orientativo | Motivo para incluirlo | Precaucion |
|---|---|---|---|
| Qwen2.5 7B Instruct | `Qwen/Qwen2.5-7B-Instruct` | Buen candidato multilingue y fuerte en instrucciones | Requiere mas memoria que un modelo de 3B |
| Llama 3.1 8B Instruct | `meta-llama/Llama-3.1-8B-Instruct` | Muy citado y con amplia bibliografia y comunidad | Puede requerir aceptar condiciones de acceso y revisar la licencia |
| Gemma 2 9B Instruct | `google/gemma-2-9b-it` | Buen candidato para comparar calidad y eficiencia | Es el mayor de la lista y tiene condiciones propias de uso |
| Mistral 7B Instruct | `mistralai/Mistral-7B-Instruct-v0.3` | Alternativa conocida y adecuada como referencia | El rendimiento en español debe medirse, no asumirse |

Para la primera ejecucion se recomienda comenzar con una variante pequena, por ejemplo Qwen2.5 3B, y comprobar el pipeline completo antes de descargar modelos de 7B, 8B o 9B. La seleccion experimental debe registrar el identificador exacto, la revision, la cuantizacion, el tokenizer, la memoria utilizada y la licencia de los pesos.

La comparacion mas limpia con el notebook de Ollama es utilizar el mismo modelo y una configuracion equivalente en ambos runtimes. Una segunda comparacion puede estudiar el efecto del modelo, pero debe presentarse separada del efecto de utilizar Ollama o Transformers.

## 3. Por que falla el codigo de ejemplo

El fragmento recibido es un resumen conceptual, no un notebook ejecutable sin cambios. Los errores mas habituales son los siguientes.

### 3.1 Whisper no esta instalado o no es el paquete correcto

```python
import whisper
```

Esta importacion requiere un paquete concreto, normalmente `openai-whisper`. El nombre del paquete instalado y el nombre que se importa no siempre coinciden. En este proyecto ya se utiliza `faster-whisper`, que tiene otra API:

```python
from faster_whisper import WhisperModel

stt = WhisperModel("base", device="cpu", compute_type="int8")
segments, _ = stt.transcribe("audio.wav", language="es", vad_filter=True)
text = " ".join(segment.text.strip() for segment in segments).strip()
```

No se deben mezclar `whisper.load_model(...)` y `WhisperModel(...)` como si fueran la misma clase.

### 3.2 Faltan dependencias de Transformers y PyTorch

El bloque del LLM necesita, como minimo, `transformers`, `torch`, `accelerate` y suficiente memoria para los pesos y la inferencia. Si aparece `ModuleNotFoundError`, la dependencia falta en el kernel seleccionado. Si aparece un error de memoria, el modelo es demasiado grande para la RAM o VRAM disponible.

Un modelo de 7B no es equivalente al modelo local de 3B usado en Ollama. En precision completa puede requerir mucha memoria; la cuantizacion puede reducirla, pero necesita una configuracion compatible con el sistema.

### 3.3 El uso de Qwen debe aplicar su plantilla de chat

La llamada `tokenizer(prompt, ...)` puede funcionar, pero no utiliza necesariamente el formato de conversacion recomendado por un modelo instruct. Es preferible usar `apply_chat_template` y decodificar solo los tokens nuevos:

```python
messages = [
    {"role": "system", "content": "Responde en espanol y de forma concisa."},
    {"role": "user", "content": text},
]

prompt = tokenizer.apply_chat_template(
    messages,
    tokenize=False,
    add_generation_prompt=True,
)
inputs = tokenizer(prompt, return_tensors="pt").to(model.device)
outputs = model.generate(**inputs, max_new_tokens=120)
new_tokens = outputs[0, inputs["input_ids"].shape[1]:]
response = tokenizer.decode(new_tokens, skip_special_tokens=True).strip()
```

Si se decodifica `outputs[0]` completo, la respuesta puede incluir de nuevo el prompt del usuario.

### 3.4 `F5TTS` no tiene esa API en todas las instalaciones

El ejemplo presupone que existe:

```python
from f5_tts import F5TTS
tts = F5TTS()
audio = tts.generate(response)
```

Esa interfaz no debe darse por válida sin comprobar la version concreta de F5-TTS. El proyecto puede cambiar sus clases, argumentos, modelos de referencia y formato de salida. Por eso, para el primer notebook conviene conservar Piper, que ya esta incluido en el proyecto y permite aislar la comparacion del LLM.

Ademas, un objeto de audio no siempre es directamente bytes WAV. Puede ser un tensor o un array que deba guardarse con una frecuencia de muestreo concreta. Escribirlo con `open(..., "wb")` solo es correcto si la API devuelve bytes WAV ya serializados.

### 3.5 `audio.wav` debe existir y tener un formato compatible

El archivo de entrada debe estar en la ruta del notebook, o debe utilizarse una ruta absoluta o `pathlib.Path`. Tambien pueden aparecer errores por frecuencia de muestreo, canales o permisos de acceso al microfono.

## 4. Instalacion orientativa

En el entorno virtual del proyecto se puede instalar la base del experimento con:

```powershell
python -m pip install transformers accelerate sentencepiece
```

`torch` debe instalarse siguiendo la variante adecuada para CPU o para la GPU disponible. No conviene fijar una orden de instalacion de CUDA sin conocer primero el hardware y la version de los controladores.

Para verificar que el notebook usa el mismo entorno donde se instalaron los paquetes:

```python
import sys
print(sys.executable)
```

La ruta debe corresponder al kernel seleccionado en Jupyter. Este es un origen frecuente de errores: el paquete se instala en un Python y el notebook se ejecuta con otro.

## 5. Carga controlada del modelo

El notebook deberia permitir cambiar el modelo sin modificar toda la cadena:

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

MODEL_NAME = "Qwen/Qwen2.5-3B-Instruct"

tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)
model = AutoModelForCausalLM.from_pretrained(
    MODEL_NAME,
    torch_dtype="auto",
    device_map="auto",
)
```

El modelo se descarga la primera vez y queda almacenado en la cache local de Hugging Face. Para un equipo con recursos limitados se debe empezar por un modelo pequeño. El modelo elegido debe registrarse junto con su version, licencia, cuantizacion, memoria utilizada y hardware.

## 6. Flujo experimental del futuro notebook

El notebook se construira por etapas para poder localizar los errores:

1. Verificar el interprete, PyTorch, Transformers y el dispositivo disponible.
2. Cargar el tokenizer y el LLM.
3. Probar una pregunta de texto sin audio.
4. Transcribir un WAV de prueba con `faster-whisper`.
5. Generar una respuesta con el historial de conversacion.
6. Sintetizarla usando Piper, que ya forma parte de la linea base local.
7. Reproducir o guardar el WAV resultante.
8. Medir tiempo de carga, tiempo hasta el primer resultado y tiempo total.

La prueba de texto de la etapa 3 es el chequeo mas barato: si falla, el problema esta en el entorno, el modelo, el tokenizer o la memoria, y todavia no tiene sentido depurar el audio.

## 7. Comparacion que debe hacerse

| Criterio | Ollama | Transformers directo |
|---|---|---|
| Instalacion inicial | Mas sencilla | Mas dependencias y configuracion |
| Control del modelo | Medio, mediante opciones del runtime | Alto, desde Python y PyTorch |
| Integracion en notebook | API HTTP sencilla | Integracion directa con tensores |
| Reproducibilidad experimental | Buena si se fija el modelo y opciones | Muy buena si se fija todo el entorno |
| Diagnostico de memoria | Mas abstracto | Visible y configurable |
| Cambio de modelos | Muy sencillo entre modelos compatibles | Sencillo, pero requiere revisar arquitectura y memoria |
| Mantenimiento de un servicio | Requiere Ollama ejecutandose | No requiere servidor aparte |
| Adecuado para el TFM | Linea base estable | Linea experimental y de investigacion |

La recomendacion es conservar Ollama como linea base reproducible y crear el notebook de Transformers como experimento controlado. Asi se puede estudiar si el control adicional compensa la complejidad, sin presentar una conclusion previa de que una tecnologia es siempre superior.

## 8. Fuentes para la seleccion de TTS

Las siguientes afirmaciones proceden de la documentacion oficial de los proyectos, pero deben interpretarse con cuidado:

| Modelo | Que respalda la fuente oficial | Redaccion adecuada para el TFM |
|---|---|---|
| F5-TTS | El repositorio describe mejoras de rendimiento y publica un benchmark de inferencia bajo condiciones concretas, por ejemplo en una GPU NVIDIA L20 | "Candidato orientado a sintesis natural y de baja latencia; su velocidad debe medirse en nuestro hardware" |
| Fish Speech 1.5 | La model card indica soporte para español y otros idiomas, y describe el modelo como TTS multilingue | "Candidato multilingue; la naturalidad conversacional debe evaluarse con pruebas propias" |
| Bark | El repositorio describe generacion de habla, risas, suspiros, musica, ruido y efectos; tambien advierte que no es un TTS convencional | "Candidato expresivo para experimentar, pero menos adecuado como linea base de dialogo estable" |

Fuentes primarias consultadas:

- F5-TTS: [repositorio oficial](https://github.com/SWivid/F5-TTS), [model card](https://huggingface.co/SWivid/F5-TTS) y articulo [F5-TTS](https://arxiv.org/abs/2410.06885).
- Fish Speech 1.5: [repositorio oficial](https://github.com/fishaudio/fish-speech), [model card](https://huggingface.co/fishaudio/fish-speech-1.5) y articulo [Fish-Speech](https://arxiv.org/abs/2411.01156).
- Bark: [repositorio oficial](https://github.com/suno-ai/bark), [model card](https://huggingface.co/suno/bark) y [documentacion de Transformers](https://huggingface.co/docs/transformers/model_doc/bark).

La fecha de consulta y las versiones deben anotarse cuando se cierre el experimento. Las licencias tampoco son identicas: el codigo de F5-TTS se publica bajo MIT, pero sus modelos preentrenados indican CC-BY-NC; Fish Speech 1.5 indica CC-BY-NC-SA-4.0; Bark indica MIT. Hay que revisar siempre la licencia exacta de la version y de los modelos descargados.

## 8. Limitaciones

- La licencia del codigo, los pesos, el tokenizer, las voces y los modelos auxiliares debe revisarse por separado.
- La calidad y latencia dependen del hardware y de la cuantizacion, no solo de la libreria utilizada.
- El pipeline modular introduce latencia entre STT, LLM y TTS y no debe confundirse con Speech-to-Speech nativo.
- Las versiones de `transformers`, PyTorch, F5-TTS y los modelos pueden cambiar sus APIs. El notebook debe guardar las versiones usadas.
