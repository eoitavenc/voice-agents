# Lesson 4: agente de voz

## 1. Que hace Lesson 4

`Version_online_agente_voz.ipynb` construye un agente conversacional de voz en tiempo real. El usuario habla desde el navegador y recibe una respuesta hablada.

El flujo general es:

```text
Microfono del usuario
        |
        v
LiveKit recibe el audio
        |
        v
Silero VAD detecta el turno de voz
        |
        v
STT convierte voz en texto
        |
        v
LLM interpreta el texto y genera una respuesta
        |
        v
TTS convierte la respuesta en audio
        |
        v
LiveKit reproduce el audio al usuario
```

En la version original, los servicios principales son:

- LiveKit para la comunicacion de audio en tiempo real.
- Silero VAD para detectar actividad de voz.
- OpenAI STT para transcribir el audio.
- OpenAI `gpt-4o` como modelo de lenguaje.
- ElevenLabs TTS para generar la voz de salida.
- Jupyter para ejecutar el agente desde el notebook.

## 2. Estructura del notebook

| Paso | Funcion | Resultado |
|---|---|---|
| 1. Importar modulos | Carga LiveKit, OpenAI, ElevenLabs y Silero | Deja disponibles las dependencias |
| 2. Configurar variables | Lee las claves y URLs desde `.env` | Prepara el entorno |
| 3. Definir `Assistant` | Crea el agente y conecta STT, LLM, TTS y VAD | Define el comportamiento del agente |
| 4. Definir `entrypoint` | Conecta el agente a una sala de LiveKit | Inicia una sesion de voz |
| 5. Ejecutar la aplicacion | Usa `WorkerOptions` y `jupyter.run_app` | Levanta el agente dentro del notebook |
| 6. Mostrar la sala | Inserta la interfaz de LiveKit en Jupyter | Permite activar audio y microfono |
| 7. Probar voces | Cambia el identificador de ElevenLabs | Permite seleccionar otra voz |

## 3. Objetos principales

### Configuracion y entorno

| Objeto | Tipo o procedencia | Funcion |
|---|---|---|
| `load_dotenv` | `python-dotenv` | Carga las variables del archivo `.env` |
| `ELEVENLABS_VOICE_ID` | Variable de Python | Identifica la voz usada por ElevenLabs |
| `OPENAI_API_KEY` | Variable de entorno | Autoriza el uso de OpenAI |
| `ELEVEN_API_KEY` | Variable de entorno | Autoriza el uso de ElevenLabs |
| `LIVEKIT_URL` | Variable de entorno | Direccion del servidor LiveKit |
| `LIVEKIT_API_KEY` | Variable de entorno | Clave publica de LiveKit |
| `LIVEKIT_API_SECRET` | Variable de entorno | Secreto de LiveKit |
| `LIVEKIT_JUPYTER_URL` | Variable de entorno | URL alternativa para obtener un token de conexion |

### Objetos de LiveKit

| Objeto | Procedencia | Funcion |
|---|---|---|
| `Agent` | `livekit.agents` | Clase base de un agente conversacional |
| `Assistant` | Clase definida en el notebook | Implementa el agente concreto |
| `AgentSession` | `livekit.agents` | Coordina una conversacion completa |
| `JobContext` | `livekit.agents` | Contiene el contexto de ejecucion y la sala |
| `WorkerOptions` | `livekit.agents` | Configura el proceso que ejecuta el agente |
| `AutoSubscribe` | `livekit.agents` | Decide que pistas de la sala se reciben |
| `RoomInputOptions` | `livekit.agents.voice.room_io` | Activa la entrada de audio de la sala |
| `entrypoint` | Funcion definida en el notebook | Punto de entrada del proceso de LiveKit |
| `jupyter.run_app` | `livekit.agents.jupyter` | Ejecuta el agente desde Jupyter |
| `room_html` | `livekit.rtc.jupyter` | Genera la interfaz HTML de la sala |

### Plugins de inteligencia artificial

| Objeto | Procedencia | Funcion |
|---|---|---|
| `openai.LLM` | `livekit.plugins.openai` | Genera respuestas a partir del texto |
| `openai.STT` | `livekit.plugins.openai` | Transcribe la voz del usuario |
| `elevenlabs.TTS` | `livekit.plugins.elevenlabs` | Sintetiza la respuesta como voz |
| `silero.VAD` | `livekit.plugins.silero` | Detecta los momentos de habla y silencio |

## 4. La clase `Assistant`

La clase `Assistant` es el objeto que representa al agente de voz. Hereda de `Agent` y configura sus capacidades:

```python
class Assistant(Agent):
    def __init__(self) -> None:
        llm = openai.LLM(model="gpt-4o")
        stt = openai.STT()
        tts = elevenlabs.TTS(
            voice_id=ELEVENLABS_VOICE_ID,
            api_key=ELEVEN_API_KEY,
        )
        vad = silero.VAD.load()

        super().__init__(
            instructions="You are a helpful assistant communicating via voice",
            stt=stt,
            llm=llm,
            tts=tts,
            vad=vad,
        )
```

El constructor crea cuatro objetos especializados y los entrega a `Agent`:

- `llm`: es el cerebro textual. Interpreta el mensaje y redacta la respuesta.
- `stt`: recibe audio y devuelve texto.
- `tts`: recibe texto y devuelve audio.
- `vad`: identifica cuando el usuario esta hablando.
- `instructions`: establece las instrucciones generales del agente.

El agente no es solamente el modelo de lenguaje. Es la combinacion de estos componentes coordinados por `AgentSession`.

## 5. El objeto `AgentSession`

`AgentSession` coordina los turnos de conversacion. Cuando se inicia con `session.start`, recibe:

```python
session = AgentSession()

await session.start(
    room=ctx.room,
    agent=Assistant(),
    room_input_options=RoomInputOptions(audio_enabled=True),
)
```

Sus responsabilidades son:

1. Recibir el audio de la sala.
2. Pasar el audio al detector VAD.
3. Enviar cada turno de voz al componente STT.
4. Entregar el texto al LLM.
5. Pasar la respuesta del LLM al componente TTS.
6. Publicar el audio resultante en la sala.

## 6. La funcion `entrypoint`

`entrypoint` es la funcion que LiveKit ejecuta para iniciar una instancia del agente:

```python
async def entrypoint(ctx: JobContext):
    await ctx.connect(auto_subscribe=AutoSubscribe.AUDIO_ONLY)

    session = AgentSession()

    await session.start(
        room=ctx.room,
        agent=Assistant(),
        room_input_options=RoomInputOptions(audio_enabled=True),
    )
```

Primero conecta el proceso con la sala. Despues crea una sesion y le asigna una instancia de `Assistant`.

## 7. Como se inicia el agente

El notebook configura un trabajador de LiveKit:

```python
worker_options = WorkerOptions(
    entrypoint_fnc=entrypoint,
    num_idle_processes=1,
)

jupyter.run_app(worker_options)
```

`WorkerOptions` indica que `entrypoint` es la funcion que debe ejecutarse. `num_idle_processes=1` mantiene un proceso disponible para atender la aplicacion.

La interfaz del navegador se prepara con:

```python
room = room_html(url, token, width="100%", height="110px")
```

El notebook modifica el `iframe` para permitir:

- `microphone` para capturar la voz.
- `autoplay` para reproducir las respuestas.

## 8. Modelo de agente de voz

El modelo de agente de voz utilizado es un modelo por turnos. Cada intervencion del usuario atraviesa una cadena de procesamiento:

### Entrada

El usuario pulsa el microfono y habla. LiveKit transporta el audio al agente.

### Deteccion

Silero VAD distingue la voz del silencio. Esto permite saber cuando puede comenzar la transcripcion y cuando termina el turno del usuario.

### Comprension

El componente STT transforma el audio en texto. Ese texto se envia al LLM junto con las instrucciones del agente y el historial de conversacion que gestione la sesion.

### Razonamiento y respuesta

El LLM genera una respuesta textual. En la version original se utiliza `gpt-4o`.

### Salida

ElevenLabs convierte la respuesta textual en audio y LiveKit la publica para que el usuario la escuche.

## 9. Sustitucion del LLM por Ollama

Ollama puede sustituir al componente LLM porque ejecuta un modelo local y ofrece una API HTTP en:

```text
http://127.0.0.1:11434
```

En este proyecto se utiliza:

```text
qwen2.5:3b
```

La prueba independiente de Ollama es:

```python
import requests

response = requests.post(
    "http://127.0.0.1:11434/api/chat",
    json={
        "model": "qwen2.5:3b",
        "messages": [
            {"role": "user", "content": "Di hola en espanol."}
        ],
        "stream": False,
    },
    timeout=120,
)
response.raise_for_status()
print(response.json()["message"]["content"])
```

Para incorporarlo a LiveKit se necesita un plugin o adaptador compatible con la interfaz LLM de la version de LiveKit instalada. El cambio afecta al cerebro textual, pero no elimina automaticamente STT, TTS, VAD ni LiveKit.

Una primera arquitectura hibrida seria:

```text
LiveKit + Silero VAD + OpenAI STT + Ollama + ElevenLabs TTS
```

Una arquitectura completamente local sustituiria tambien el reconocimiento y la sintesis:

```text
LiveKit o transporte local + Silero VAD + faster-whisper + Ollama + TTS local
```

## 10. Resumen

Lesson 4 enseña a ensamblar un agente de voz a partir de componentes especializados. La clase `Assistant` define las capacidades del agente; `AgentSession` coordina la conversacion; `entrypoint` conecta el agente con LiveKit; y `jupyter.run_app` permite ejecutar todo desde el notebook.

Ollama puede ocupar el lugar del LLM de OpenAI. Para obtener un agente completamente local tambien hay que reemplazar OpenAI STT y ElevenLabs TTS, mientras que Silero VAD ya puede ejecutarse localmente.
