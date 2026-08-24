# Lesson 4 con librerias open source

## 1. Objetivo

Esta guia describe como construir un agente de voz equivalente al de `Version_online_agente_voz.ipynb`, pero utilizando componentes de codigo abierto y modelos ejecutados localmente.

La arquitectura propuesta es:

```text
Microfono
    |
    v
Transporte de audio local o LiveKit autohospedado
    |
    v
Silero VAD
    |
    v
faster-whisper (STT)
    |
    v
Ollama + modelo local (LLM)
    |
    v
Piper (TTS)
    |
    v
Altavoces
```

A diferencia de la version original de Lesson 4, esta arquitectura no necesita claves de OpenAI ni de ElevenLabs. Los datos de voz y texto pueden permanecer en el equipo.

## 2. Sustitucion de componentes

| Funcion | Lesson 4 original | Alternativa open source/local | Resultado |
|---|---|---|---|
| Transporte de audio | LiveKit Cloud o servicio LiveKit | LiveKit autohospedado o audio local | Comunicacion sin depender de un proveedor externo |
| Deteccion de voz | Silero VAD | Silero VAD | Detecta el inicio y el final de cada turno |
| STT | OpenAI STT | `faster-whisper` | Transcribe audio localmente |
| LLM | OpenAI `gpt-4o` | Ollama + `qwen2.5:3b` | Genera la respuesta localmente |
| TTS | ElevenLabs | Piper | Sintetiza voz localmente |
| Orquestacion | LiveKit Agents | Python y, opcionalmente, LiveKit Agents | Coordina el ciclo de conversacion |
| Interfaz | Panel LiveKit en Jupyter | Interfaz propia, Jupyter o LiveKit | Permite grabar y reproducir audio |

## 3. Que significa open source en este proyecto

Hay que distinguir entre tres elementos:

1. **Codigo fuente**: la libreria que ejecuta el procesamiento.
2. **Pesos del modelo**: los archivos aprendidos que utiliza la libreria.
3. **Voces o datos**: modelos de voz y datasets que pueden tener una licencia diferente.

Que una libreria sea open source no significa automaticamente que todos sus modelos tengan la misma licencia. Antes de distribuir una aplicacion hay que revisar la licencia concreta de cada modelo de Whisper, Qwen y Piper.

## 4. Librerias propuestas

### 4.1 `faster-whisper`

`faster-whisper` ejecuta modelos Whisper mediante CTranslate2. Se utiliza para convertir el audio del usuario en texto sin enviar el archivo a una API externa.

Ventajas:

- Ejecucion local.
- Buen soporte para varios idiomas.
- Puede utilizar CPU o GPU.
- Permite elegir modelos de distinto tamaño.

El modelo `small` o `base` suele ser un punto de partida razonable. El modelo mayor mejora la transcripcion, pero necesita mas memoria y tiempo.

### 4.2 Silero VAD

Silero VAD identifica si un fragmento de audio contiene voz. Evita enviar continuamente silencio al sistema STT y ayuda a separar los turnos de la conversacion.

### 4.3 Ollama

Ollama ejecuta el LLM en local y ofrece una API HTTP:

```text
http://127.0.0.1:11434
```

En el proyecto se utiliza:

```text
qwen2.5:3b
```

El modelo puede sustituirse por otro compatible con el hardware disponible. Los pesos del modelo tienen que revisarse de forma independiente, porque la licencia del modelo no tiene por que coincidir con la de Ollama.

### 4.4 Piper

Piper es un sintetizador de voz local. Recibe texto y genera un archivo de audio WAV. Las voces se descargan por separado y cada voz puede tener sus propias condiciones de uso.

### 4.5 LiveKit

LiveKit puede utilizarse como transporte de audio en tiempo real. Si se quiere evitar un servicio gestionado, es posible autohospedar el servidor. Para una primera prueba local tambien se puede prescindir de LiveKit y trabajar directamente con el microfono y los altavoces del equipo.

## 5. Instalacion orientativa

La instalacion depende del sistema operativo y de si se utiliza CPU o GPU. Como referencia para el entorno virtual de este proyecto:

```powershell
python -m pip install faster-whisper silero-vad requests sounddevice soundfile
```

Para Piper se debe instalar el paquete disponible para el sistema y descargar una voz compatible. El nombre exacto del paquete puede cambiar entre versiones, por lo que conviene consultar la documentacion del proyecto antes de fijar el comando en produccion.

Ollama se instala fuera de Python. El modelo utilizado se descarga con:

```powershell
ollama pull qwen2.5:3b
```

Comprobacion basica:

```powershell
ollama --version
ollama list
```

## 6. Objetos del agente open source

| Objeto | Funcion |
|---|---|
| `AudioInput` | Captura bloques de audio del microfono |
| `VAD` | Decide si un bloque contiene voz |
| `WhisperTranscriber` | Convierte un turno de audio en texto |
| `OllamaClient` | Envia el texto a Ollama y obtiene la respuesta |
| `PiperSynthesizer` | Convierte la respuesta en audio |
| `AudioOutput` | Reproduce el audio por los altavoces |
| `VoiceAgent` | Coordina todos los componentes |
| `ConversationState` | Conserva instrucciones e historial de mensajes |

Estos nombres representan las responsabilidades de la arquitectura. Se pueden implementar como clases propias o sustituir por los objetos equivalentes de la libreria de orquestacion seleccionada.

## 7. Cliente LLM para Ollama

El componente LLM puede ser un cliente pequeno basado en la API local:

```python
import requests


class OllamaClient:
    def __init__(
        self,
        model: str = "qwen2.5:3b",
        base_url: str = "http://127.0.0.1:11434",
    ) -> None:
        self.model = model
        self.base_url = base_url.rstrip("/")

    def chat(self, messages: list[dict[str, str]]) -> str:
        response = requests.post(
            f"{self.base_url}/api/chat",
            json={
                "model": self.model,
                "messages": messages,
                "stream": False,
                "options": {
                    "temperature": 0.2,
                    "num_predict": 120,
                },
            },
            timeout=120,
        )
        response.raise_for_status()
        return response.json()["message"]["content"].strip()
```

El estado de la conversacion puede conservarse con una lista de mensajes:

```python
conversation = [
    {
        "role": "system",
        "content": "Eres un asistente de voz util. Responde en espanol y de forma concisa.",
    }
]

conversation.append({"role": "user", "content": "Hola, ¿que puedes hacer?"})
answer = ollama_client.chat(conversation)
conversation.append({"role": "assistant", "content": answer})
```

## 8. Transcripcion local con faster-whisper

Una implementacion sencilla del componente STT es:

```python
from faster_whisper import WhisperModel


class WhisperTranscriber:
    def __init__(self, model_size: str = "base") -> None:
        self.model = WhisperModel(
            model_size,
            device="cpu",
            compute_type="int8",
        )

    def transcribe(self, audio_path: str) -> str:
        segments, _ = self.model.transcribe(
            audio_path,
            language="es",
            vad_filter=True,
        )
        return " ".join(segment.text.strip() for segment in segments).strip()
```

En equipos con una GPU compatible se pueden cambiar `device` y `compute_type` para reducir la latencia. La configuracion debe probarse con el hardware real del proyecto.

## 9. Sintesis local con Piper

Piper normalmente recibe texto y genera un archivo WAV. El agente puede encapsular el comando o la API disponible en la instalacion:

```python
import subprocess


class PiperSynthesizer:
    def __init__(
        self,
        voice_name: str = "es_ES-davefx-medium",
        data_dir: str = "models/piper",
    ) -> None:
        self.voice_name = voice_name
        self.data_dir = data_dir

    def synthesize(self, text: str, output_path: str) -> str:
        subprocess.run(
            [
                "piper",
                "--model",
                self.voice_name,
                "--data-dir",
                self.data_dir,
                "--output_file",
                output_path,
            ],
            input=text,
            text=True,
            check=True,
        )
        return output_path
```

El comando puede variar segun la instalacion de Piper. La voz elegida debe descargarse previamente y su licencia debe ser compatible con el uso previsto.

## 10. Orquestador `VoiceAgent`

El objeto `VoiceAgent` representa el agente completo. Su responsabilidad es coordinar los componentes, no implementar el reconocimiento, el razonamiento o la sintesis desde cero.

```python
class VoiceAgent:
    def __init__(self, transcriber, llm, synthesizer) -> None:
        self.transcriber = transcriber
        self.llm = llm
        self.synthesizer = synthesizer
        self.messages = [
            {
                "role": "system",
                "content": "Eres un asistente de voz util. Responde en espanol y de forma concisa.",
            }
        ]

    def process_turn(self, audio_path: str, output_path: str) -> str:
        user_text = self.transcriber.transcribe(audio_path)
        if not user_text:
            return ""

        self.messages.append({"role": "user", "content": user_text})
        answer = self.llm.chat(self.messages)
        self.messages.append({"role": "assistant", "content": answer})
        self.synthesizer.synthesize(answer, output_path)
        return answer
```

El VAD y el transporte de audio rodean a `process_turn`. El VAD decide cuando hay un turno completo y el sistema de audio entrega el archivo temporal al transcriptor.

## 11. Modelo de funcionamiento por turnos

En una implementacion por turnos, el agente sigue este ciclo:

```text
1. Capturar audio
2. Esperar a que VAD detecte voz
3. Acumular audio mientras el usuario habla
4. Detectar silencio y cerrar el turno
5. Transcribir con faster-whisper
6. Anadir el texto al historial
7. Generar una respuesta con Ollama
8. Sintetizar la respuesta con Piper
9. Reproducir el WAV
10. Volver a escuchar
```

Este modelo es sencillo de depurar y adecuado para una primera version. No es completamente simultaneo: normalmente espera a tener el turno completo antes de responder.

## 12. Arquitecturas posibles

### Opcion A: prueba local sencilla

```text
sounddevice + Silero VAD + faster-whisper + Ollama + Piper
```

Es la opcion recomendada para validar el procesamiento de voz sin introducir todavia salas, tokens o navegadores.

### Opcion B: navegador y servidor propio

```text
Navegador + LiveKit autohospedado + agente Python local
```

Permite conservar una experiencia similar a Lesson 4, pero requiere configurar el servidor LiveKit, la autenticacion y la interfaz web.

### Opcion C: LiveKit Agents

```text
LiveKit Agents + adaptadores locales de STT, LLM y TTS
```

Es la opcion mas cercana al notebook original. El agente debe proporcionar adaptadores que cumplan las interfaces esperadas por la version instalada de LiveKit Agents. No se debe asumir que cualquier plugin local es compatible sin comprobar su API.

## 13. Ventajas y limitaciones

### Ventajas

- No requiere claves de OpenAI ni ElevenLabs.
- Los audios y textos pueden permanecer en el equipo.
- Permite controlar los modelos y la configuracion.
- Facilita experimentar con distintos modelos locales.
- Reduce el coste por peticion.

### Limitaciones

- La latencia depende mucho de CPU, GPU y memoria.
- Un modelo pequeno como `qwen2.5:3b` puede razonar peor que `gpt-4o`.
- La instalacion local requiere mas configuracion.
- La calidad de la voz depende del modelo Piper elegido.
- Las licencias de los pesos y voces deben revisarse por separado.
- LiveKit autohospedado necesita administracion y recursos propios.

## 14. Orden recomendado de implementacion

1. Verificar Ollama con una peticion de texto.
2. Probar `faster-whisper` con un archivo WAV.
3. Probar Piper con una frase fija.
4. Integrar STT, Ollama y TTS en un flujo por turnos.
5. Incorporar Silero VAD para detectar automaticamente los turnos.
6. Medir tiempo de transcripcion, generacion y sintesis.
7. Anadir la captura directa del microfono.
8. Integrar LiveKit solo si se necesita comunicacion desde el navegador.

## 15. Conclusion

Un agente de voz open source se construye separando cinco responsabilidades: entrada de audio, deteccion de voz, transcripcion, generacion de texto y sintesis de voz.

La alternativa local a Lesson 4 puede utilizar `faster-whisper`, Silero VAD, Ollama y Piper. LiveKit es opcional para una prueba local y puede autohospedarse cuando se necesita una interfaz de navegador o comunicacion en tiempo real.

La arquitectura minima recomendada para este proyecto es:

```text
Silero VAD + faster-whisper + Ollama qwen2.5:3b + Piper
```
