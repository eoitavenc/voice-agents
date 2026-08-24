# Plan para un nuevo agente Speech-to-Speech

## 1. Objetivo

Crear un notebook independiente para evaluar un agente de voz en tiempo real. El agente debe recibir audio, mantener una conversacion y devolver audio con la menor latencia posible.

Este proyecto sera independiente de `Version_online_agente_voz.ipynb` y `Ollama_Local_Agent.ipynb`.

## 2. Dos arquitecturas posibles

### 2.1 Speech-to-Speech nativo

El modelo recibe audio y genera audio directamente:

```text
Microfono -> transporte realtime -> modelo audio-audio -> transporte realtime -> altavoces
```

El modelo puede trabajar con informacion prosodica, pausas e interrupciones sin necesitar una transcripcion textual como paso principal.

### 2.2 Pipeline de voz en tiempo real

Se mantienen componentes independientes, pero todos trabajan con streaming:

```text
Microfono
    -> VAD
    -> STT por streaming
    -> LLM por streaming
    -> TTS por streaming
    -> altavoces
```

Esta arquitectura no es Speech-to-Speech nativa, pero suele ser mas facil de depurar, sustituir y ejecutar localmente.

## 3. Opcion comercial recomendada: OpenAI Realtime

### 3.1 OpenAI Realtime API + LiveKit

```text
Navegador
    -> WebRTC / LiveKit
    -> OpenAI Realtime API
    -> LiveKit / WebRTC
    -> navegador
```

Componentes:

| Componente | Tecnologia | Funcion |
|---|---|---|
| Cliente y transporte | WebRTC o LiveKit | Audio bidireccional de baja latencia |
| Modelo realtime | OpenAI Realtime API | Entrada y salida de audio |
| Orquestacion | LiveKit Agents o SDK del proveedor | Sesiones, herramientas y eventos |
| Configuracion | `python-dotenv` | Variables y claves fuera del notebook |

Ventajas:

- Integracion directa con una experiencia Speech-to-Speech.
- Streaming bidireccional.
- Interrupciones y turn-taking gestionados por la API.
- Menos componentes que mantener.
- Buena opcion para comparar con el agente online actual.

Inconvenientes:

- Coste por uso.
- Dependencia de un proveedor.
- El audio y el contexto se procesan fuera del equipo.
- Se deben revisar precios, modelos y limites en la documentacion oficial antes de fijar la configuracion.

### 3.2 Alternativas comerciales

| Opcion | Enfoque | Cuando elegirla |
|---|---|---|
| Gemini Live API | Audio bidireccional en tiempo real | Si el proyecto ya utiliza servicios de Google |
| ElevenLabs Conversational AI | Agente conversacional con voces naturales | Si la calidad de voz es prioritaria |
| Azure Voice Live | Voz realtime orientada a entornos empresariales | Si se necesita integracion con Azure |
| Deepgram Voice Agent | Agente de voz con servicios integrados | Si se prioriza STT streaming y latencia |

Para este proyecto, OpenAI Realtime es la primera opcion comercial recomendada. Gemini Live es la alternativa mas interesante para comparar otro proveedor con audio bidireccional. ElevenLabs Conversational AI es preferible si la prioridad es la naturalidad de la voz. Azure Voice Live y Deepgram Voice Agent son opciones validas de agente realtime, pero deben considerarse como plataformas integradas y no necesariamente como un unico modelo S2S nativo.

Los nombres de modelos, APIs, precios y compatibilidades cambian. El notebook debe guardar la fecha de consulta y no asumir que una version futura mantiene la misma interfaz.

## 4. Opcion open source y autocontenida

### 4.1 Opcion practica: pipeline local realtime

```text
Microfono
    -> sounddevice o WebRTC
    -> Silero VAD
    -> faster-whisper o whisper.cpp
    -> Ollama + Qwen/Llama/Mistral
    -> Piper o Kokoro
    -> altavoces
```

Componentes propuestos:

| Componente | Tecnologia | Funcion |
|---|---|---|
| Captura | `sounddevice` o WebRTC | Entrada y salida de audio |
| Transporte | LiveKit autohospedado o audio local | Comunicacion realtime |
| VAD | Silero VAD | Detectar inicio y final de turno |
| STT | `faster-whisper` o `whisper.cpp` | Transcripcion local |
| LLM | Ollama | Servir un modelo local |
| Modelo LLM | Qwen, Llama o Mistral | Generar la respuesta |
| TTS | Piper o Kokoro | Sintesis local |
| Orquestacion | Pipecat, LiveKit Agents o Python async | Coordinar streaming e interrupciones |

Esta opcion reutiliza conocimientos y dependencias del agente local actual. Para que sea realmente realtime se deben implementar transcripciones parciales, respuestas por streaming, sintesis por fragmentos e interrupcion del audio cuando comienza un nuevo turno.

### 4.2 Opcion nativa audio-audio

Se puede investigar una arquitectura basada en un modelo abierto de audio-audio, por ejemplo Moshi de Kyutai u otros modelos Omni con entrada y salida de audio. Moshi es la opcion open source recomendada para el experimento S2S nativo, pero se debe revisar por separado la licencia del codigo, los pesos, el codec de audio y cualquier modelo auxiliar antes de usarlo con fines comerciales.

```text
Microfono
    -> WebRTC o Pipecat
    -> modelo Speech-to-Speech local
    -> WebRTC o Pipecat
    -> altavoces
```

Ventajas:

- Arquitectura mas cercana a Speech-to-Speech real.
- Mayor control sobre privacidad y despliegue.
- No necesita separar obligatoriamente STT, LLM y TTS.

Riesgos:

- Mayor consumo de GPU y memoria.
- Instalacion y compatibilidad mas complejas.
- Menor variedad de voces y herramientas que en servicios comerciales.
- Se debe comprobar que el modelo elegido soporte espanol, streaming, interrupciones y el hardware disponible.
- El soporte para espanol y la integracion con el resto de la aplicacion deben validarse antes de convertirlo en la implementacion principal.

## 5. Comparacion para decidir

| Criterio | Realtime comercial | Pipeline local | Modelo audio-audio local |
|---|---|---|---|
| Latencia inicial | Baja | Media, optimizable | Potencialmente baja |
| Calidad de voz | Alta | Depende del TTS | Depende del modelo |
| Privacidad | Baja o media | Alta | Alta |
| Coste por uso | Si | No, pero requiere hardware | No, pero requiere hardware |
| Facilidad de prototipo | Alta | Media | Baja |
| Modularidad | Media | Alta | Baja o media |
| Control de modelos | Bajo | Alto | Alto |
| GPU local | No necesariamente | Recomendable | Normalmente necesaria |

## 6. Recomendacion para el proyecto

Se propone crear el nuevo notebook con una implementacion principal y dos referencias de comparacion:

### Implementacion principal: referencia comercial

Usar `OpenAI Realtime API` con WebRTC o LiveKit. Esta fase permite comprobar rapidamente:

- Latencia de respuesta.
- Calidad de la conversacion.
- Interrupciones.
- Deteccion de turnos.
- Diferencia frente al pipeline actual.

### Linea base local: pipeline modular

Usar `Pipecat` o `LiveKit Agents` como orquestador y conectar:

```text
Silero VAD -> faster-whisper -> Ollama -> Piper/Kokoro
```

Esta linea base permite medir cuanto se aproxima la experiencia local a la opcion realtime comercial. No debe describirse como S2S nativo, aunque la experiencia final sea de voz a voz.

### Experimento open source: Moshi

Moshi se utilizara como experimento de Speech-to-Speech nativo, no como dependencia obligatoria del agente principal. Se compararan latencia, calidad en espanol, interrupciones y consumo de recursos con las dos arquitecturas anteriores. Si el hardware o el soporte linguistico no son suficientes, el pipeline local modular seguira siendo la alternativa reproducible.

La recomendacion final es:

```text
Prototipo comercial: OpenAI Realtime API + LiveKit
Linea base local: faster-whisper + Ollama + Piper/Kokoro
Experimento S2S open source: Moshi + Pipecat
```

## 7. Propuesta de nuevo notebook

Nombre sugerido: `Speech_to_Speech_Realtime_Agent.ipynb`

### Seccion 1: Objetivo y arquitectura

- Explicar la diferencia entre Speech-to-Speech nativo y pipeline STT-LLM-TTS.
- Mostrar el diagrama del flujo elegido.
- Indicar si se ejecuta en nube o localmente.

### Seccion 2: Entorno y configuracion

- Comprobar version de Python.
- Comprobar microfono, altavoces y, si aplica, GPU.
- Cargar variables desde `.env`.
- No mostrar ni guardar claves en salidas del notebook.

### Seccion 3: Transporte de audio

- Configurar WebRTC, LiveKit o captura local.
- Probar entrada y salida de audio.
- Medir frecuencia de muestreo y canales.

### Seccion 4: Conexion con el modelo

- Inicializar la API realtime comercial como flujo principal.
- Mantener una implementacion de referencia con el pipeline local modular.
- Probar Moshi como flujo S2S nativo experimental.
- Configurar instrucciones, voz y modelo.
- Implementar eventos de audio, texto parcial y final de turno.

### Seccion 5: Conversacion e interrupciones

- Detectar cuando el usuario empieza a hablar.
- Cancelar la respuesta de audio en curso.
- Enviar audio por fragmentos.
- Mostrar opcionalmente transcripcion y latencia.

### Seccion 6: Prueba controlada

- Ejecutar tres preguntas en espanol.
- Probar una interrupcion mientras el agente habla.
- Registrar errores y tiempos de respuesta.
- Evitar guardar audio o credenciales por defecto.

### Seccion 7: Evaluacion

Medir como minimo:

- Latencia hasta el primer audio.
- Latencia de respuesta completa.
- Tiempo de deteccion del turno.
- Calidad de transcripcion.
- Naturalidad de la voz.
- Tasa de interrupciones gestionadas correctamente.
- Uso de CPU, RAM y GPU.
- Coste por minuto en la opcion comercial.

### Seccion 8: Conclusion

- Comparar realtime comercial, pipeline local y modelo audio-audio local.
- Documentar versiones y fecha de consulta.
- Indicar que arquitectura se recomienda para el TFM y por que.

## 8. Variables de configuracion orientativas

Para la opcion comercial:

```dotenv
OPENAI_API_KEY=
OPENAI_REALTIME_MODEL=
LIVEKIT_URL=
LIVEKIT_API_KEY=
LIVEKIT_API_SECRET=
```

Para la opcion local:

```dotenv
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=qwen2.5:3b
PIPER_MODEL_PATH=models/piper/es_ES-davefx-medium.onnx
```

Estas variables son orientativas. El archivo `.env` debe permanecer ignorado por Git.

## 9. Decision pendiente

Antes de implementar el notebook se deben fijar:

1. Confirmar el primer prototipo con OpenAI Realtime API y WebRTC/LiveKit.
2. Confirmar si el pipeline local se ejecutara con microfono local o con LiveKit autohospedado.
3. Comprobar licencias, soporte para espanol y requisitos de Moshi.
4. Medir el hardware disponible para las alternativas locales.
5. Fijar las metricas y la fecha de consulta de precios y versiones.
