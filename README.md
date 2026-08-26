# Agentes de voz

Proyecto del TFM para experimentar con agentes de voz usando una versión online basada en LiveKit y una versión local con Ollama, reconocimiento de voz y síntesis de voz.

## Requisitos

- Python 3.12 (probado con Python 3.12.10)
- Windows, macOS o Linux
- Micrófono y altavoces
- [Ollama](https://ollama.com/) para el pipeline local modular
- El modelo de voz Piper incluido en `models/piper/`, o una copia descargada localmente
- Una GPU CUDA es recomendable para los modelos S2S; el entorno actual `voiceagent` usa PyTorch CPU

## Instalación

Se recomienda crear un entorno virtual nuevo. No es necesario subir ni reutilizar la carpeta `voiceagent/` del proyecto, ya que contiene un entorno virtual local.

```bash
python -m venv .venv
```

En Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

En macOS o Linux:

```bash
source .venv/bin/activate
```

Instala primero la base común:

```bash
python -m pip install --upgrade pip
python -m pip install torch torchaudio sounddevice soundfile numpy requests python-dotenv jupyter ipykernel
```

En Windows, si usas el entorno incluido en este proyecto, ejecuta los comandos anteriores con `voiceagent\Scripts\python.exe` o activa antes `voiceagent`.

### Pipeline local modular

Para `faster-whisper`, Silero VAD, Ollama y Piper:

```powershell
python -m pip install faster-whisper silero-vad piper-tts
```

Instala Ollama aparte y descarga el modelo local:

```powershell
ollama pull qwen2.5:3b
```

El pipeline modular no es un modelo Speech-to-Speech nativo: utiliza `STT -> LLM -> TTS`.

### Qwen2.5-Omni-3B

Para el notebook `speech-to-speech/Speech_to_Speech_Qwen2.5_Omni_3B.ipynb` instala las dependencias open source:

```powershell
python -m pip install transformers accelerate soundfile qwen-omni-utils audioread
```

El modelo se descargará de Hugging Face la primera vez que se ejecute la celda de carga. No se necesitan claves ni una cuenta de pago. Se recomienda una GPU CUDA con memoria suficiente; en CPU la inferencia puede ser demasiado lenta o no viable.

### MiniCPM-o 4.5

El notebook [Speech_to_Speech_MiniCPM_o_4_5.ipynb](speech-to-speech/Speech_to_Speech_MiniCPM_o_4_5.ipynb) usa la API oficial de Transformers. Para conversación hablada, instala las versiones recomendadas por la model card:

```powershell
python -m pip install "setuptools<70" "transformers==4.51.0" accelerate "torch>=2.3.0,<=2.8.0" "torchaudio<=2.8.0" "minicpmo-utils[all]>=1.0.5" librosa soundfile
```

El modelo `openbmb/MiniCPM-o-4_5` y sus pesos se descargan desde Hugging Face durante la primera ejecución. Esta opción es local y no cobra por petición, pero la inferencia PyTorch requiere preferiblemente una GPU Nvidia con memoria suficiente. Para CPU o equipos con menos memoria, revisa las variantes cuantizadas y las instrucciones oficiales de `llama.cpp` u Ollama.

### Servicios online opcionales

Solo son necesarios para los notebooks basados en LiveKit y proveedores externos. No forman parte de las implementaciones open source locales:

```powershell
python -m pip install livekit livekit-agents livekit-plugins-elevenlabs livekit-plugins-openai livekit-plugins-silero
```

Estos servicios pueden requerir cuentas, credenciales y pago por uso. No los instales si únicamente vas a ejecutar el pipeline local, Qwen2.5-Omni o MiniCPM-o.

Si se van a ejecutar los notebooks, registra el entorno como kernel de Jupyter:

```bash
python -m ipykernel install --user --name voiceagents --display-name "Python (VoiceAgents)"
```

## Configuración

Crea un archivo `.env` en la raíz del proyecto a partir de las variables necesarias para tu instalación. No subas este archivo al repositorio.

Para la versión online, normalmente se necesitan:

```dotenv
ELEVEN_API_KEY=
OPENAI_API_KEY=
LIVEKIT_URL=
LIVEKIT_API_KEY=
LIVEKIT_API_SECRET=
ELEVENLABS_VOICE_ID=
LIVEKIT_JUPYTER_URL=
```

Rellena únicamente las variables que utilice el notebook o script que vayas a ejecutar. Las claves son credenciales privadas y no deben aparecer en notebooks, capturas ni commits.

## Modelo Piper

La síntesis local utiliza el modelo español:

```text
models/piper/es_ES-davefx-medium.onnx
models/piper/es_ES-davefx-medium.onnx.json
```

El archivo `.onnx` ocupa aproximadamente 60 MB. Si no se incluye en el repositorio, debe descargarse y colocarse en esa misma ruta antes de ejecutar la versión local.

## Ejecución

Abre VS Code o Jupyter y selecciona el kernel `Python (VoiceAgents)`. Después ejecuta:

- `speech-to-text-to-speech/Version_online_agente_voz.ipynb` para la versión online.
- `speech-to-text-to-speech/Ollama_Local_Agent.ipynb` para la versión local con Ollama.
- `speech-to-text-to-speech/Transformers_Local_Voice_Agent.ipynb` para la versión local con Transformers.

Antes de ejecutar la versión local, comprueba que Ollama está instalado y que tienes descargado el modelo indicado en las celdas del notebook. El micrófono debe estar disponible para Python y para el navegador si el flujo utiliza Jupyter o LiveKit.

## Estructura

```text
VoiceAgents/
├── Docs/                         Documentación del proyecto
├── models/piper/                 Modelo y configuración de Piper
├── models/audio/                 Audios de prueba o resultados generados
├── testing/                      Matriz de pruebas de modelos y runtimes
├── speech-to-text-to-speech/     Notebooks del pipeline de voz
└── README.md
```

Los directorios `.ipynb_checkpoints/`, `__pycache__/`, `voiceagent/` y los archivos `.env` son artefactos locales y no forman parte de la instalación reproducible.
