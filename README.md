# Agentes de voz

Proyecto del TFM para experimentar con agentes de voz usando una versión online basada en LiveKit y una versión local con Ollama, reconocimiento de voz y síntesis de voz.

## Requisitos

- Python 3.12 (probado con Python 3.12.10)
- Windows, macOS o Linux
- Micrófono y altavoces
- [Ollama](https://ollama.com/) para la versión local
- Una cuenta y credenciales de los servicios online para la versión basada en LiveKit
- El modelo de voz Piper incluido en `models/piper/`, o una copia descargada localmente

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

Instala las dependencias:

```bash
python -m pip install --upgrade pip
python -m pip install livekit livekit-agents livekit-plugins-elevenlabs livekit-plugins-openai livekit-plugins-silero python-dotenv sounddevice soundfile piper-tts faster-whisper torch torchaudio jupyter ipykernel
```

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

- `Version_online_agente_voz.ipynb` para la versión online.
- `Ollama_Local_Agent.ipynb` para la versión local con Ollama.

Antes de ejecutar la versión local, comprueba que Ollama está instalado y que tienes descargado el modelo indicado en las celdas del notebook. El micrófono debe estar disponible para Python y para el navegador si el flujo utiliza Jupyter o LiveKit.

## Estructura

```text
VoiceAgents/
├── Docs/                         Documentación del proyecto
├── models/piper/                 Modelo y configuración de Piper
├── models/audio/                 Audios de prueba o resultados generados
├── Ollama_Local_Agent.ipynb      Agente local
├── Version_online_agente_voz.ipynb  Agente online
└── README.md
```

Los directorios `.ipynb_checkpoints/`, `__pycache__/`, `voiceagent/` y los archivos `.env` son artefactos locales y no forman parte de la instalación reproducible.
