# Plan para un nuevo agente Speech-to-Speech

## 1. Objetivo

Crear un notebook independiente para evaluar un agente de voz en tiempo real. El agente debe recibir audio, mantener una conversacion y devolver audio con la menor latencia posible.

Este proyecto sera independiente de `speech-to-text-to-speech/Version_online_agente_voz.ipynb` y `speech-to-text-to-speech/Ollama_Local_Agent.ipynb`.

## 2. Arquitecturas posibles

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

Esta arquitectura no es Speech-to-Speech nativa, aunque la experiencia final sea de voz a voz. Suele ser mas facil de depurar, sustituir y ejecutar localmente.

### 2.3 Criterio de clasificacion

No se debe clasificar un modelo como S2S solo porque acepte o genere audio. Para este plan se usaran estas categorias:

| Categoria | Entrada principal | Salida | Ejemplos |
|---|---|---|---|
| S2S nativo | Voz del usuario y contexto conversacional | Voz de respuesta, opcionalmente texto | Qwen2.5-Omni, Moshi, GLM-4-Voice |
| Pipeline modular | Audio -> STT -> texto -> LLM | Texto -> TTS -> audio | faster-whisper -> Ollama -> Piper |
| TTS contextual | Texto y audio de referencia opcional | Audio sintetizado | Fish Audio S2 Pro, CSM |

La clasificación se verificara para la version exacta mediante la documentación, ejemplos, formato de entrada y salida, idiomas y requisitos de hardware.

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

## 4. Opciones open source y autocontenidas

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

### 4.2 Candidatos S2S nativos

Se investigaran modelos que reciban audio conversacional y generen audio de respuesta. La primera opcion para el experimento en español sera Qwen2.5-Omni. Moshi y GLM-4-Voice se conservaran como referencias tecnicas con las limitaciones linguisticas indicadas.

| Opcion | Entrada y salida | Idiomas documentados | Tiempo real | GPU orientativa | Papel en el proyecto |
|---|---|---|---|---|---|
| Qwen2.5-Omni-3B | Audio, texto, imagen o video -> texto y audio | Multilingue, incluido español | Si, streaming | Al menos 18-22 GB teoricos en BF16 | Primera opcion S2S en Colab |
| Qwen2.5-Omni-7B GPTQ/AWQ | Audio y texto -> texto y audio | Multilingue, incluido español | Si, streaming | Aproximadamente 12-18 GB teoricos según variante | Ampliacion para GPU limitada |
| Qwen2.5-Omni-7B BF16 | Audio y texto -> texto y audio | Multilingue, incluido español | Si, streaming | Aproximadamente 31 GB teoricos; mas en la practica | Variante de mayor calidad, no adecuada para T4 de 16 GB |
| Moshi / Moshiko | Audio del usuario -> audio del agente y texto auxiliar | Principalmente ingles | Si, full-duplex | Aproximadamente 24 GB | Referencia S2S y de latencia |
| GLM-4-Voice-9B | Voz -> voz y texto | Chino e ingles | Si | CUDA; existe modo int4 | Referencia avanzada, no principal para español |
| CSM-1B | Texto y contexto de audio -> audio | Principalmente ingles | Limitado | GPU CUDA | Generador contextual, no S2S completo |
| Fish Audio S2 Pro | Texto y audio de referencia -> audio | Multilingue, incluido español | Depende del servidor | Al menos 24 GB recomendados | TTS avanzado, no S2S conversacional |

Solo las tres variantes de Qwen2.5-Omni, Moshi y GLM-4-Voice se estudiaran inicialmente como candidatos S2S nativos. CSM y Fish Audio S2 Pro se mantendran en la categoria de TTS contextual.

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

#### Qwen2.5-Omni

Qwen2.5-Omni es un modelo multimodal end-to-end que recibe audio y puede generar texto y habla. Su repositorio oficial incluye ejemplos de conversación por voz, salida de audio, conversación multivuelta y streaming.

Se evaluaran dos variantes:

- `Qwen/Qwen2.5-Omni-3B`: primera prueba en Colab, si la GPU dispone de memoria suficiente.
- `Qwen/Qwen2.5-Omni-7B-GPTQ-Int4` o `Qwen/Qwen2.5-Omni-7B-AWQ`: variante cuantizada para reducir el consumo.

El modelo 7B en BF16 no se considera adecuado para una T4 de 16 GB. La documentación oficial indica alrededor de 31 GB teoricos para BF16, con un consumo practico superior. Las variantes cuantizadas reducen el consumo, pero deben medirse en la GPU concreta. El repositorio y el modelo se publican bajo Apache-2.0.

Qwen2.5-Omni sera el candidato principal porque combina entrada y salida de audio, soporte multilingue incluido espanol, generación en streaming y una ruta reproducible mediante Transformers.

#### Moshi

Moshi es un modelo S2S full-duplex orientado al dialogo hablado en tiempo real. Modela el audio del usuario y el audio del agente en flujos paralelos, utiliza el codec Mimi y declara una latencia practica cercana a 200 ms en una GPU L4.

Su limitacion principal para este TFM es que los modelos publicados estan orientados principalmente al ingles. Se utilizara como referencia de arquitectura y latencia, no como candidato principal para la evaluacion en español. El codigo utiliza licencias MIT y Apache-2.0 segun el componente, y los pesos se publican bajo CC-BY 4.0. La implementacion PyTorch requiere una GPU con aproximadamente 24 GB.

#### GLM-4-Voice

GLM-4-Voice es un sistema de voz a voz con tokenizer de voz, modelo conversacional y decoder de voz. Ofrece conversación en tiempo real, respuesta hablada y control de emocion, entonacion, velocidad y dialecto.

La documentación oficial declara soporte directo para chino e ingles, no para español. Se considerara un candidato experimental y una referencia de diseño, pero no se incluira en la comparacion principal en español salvo que una prueba especifica demuestre soporte suficiente. El codigo usa Apache-2.0 y los pesos tienen condiciones propias.

### 4.3 Modelos de audio que no son S2S nativo

#### Fish Audio S2 Pro

Fish Audio S2 Pro es un TTS multilingue avanzado con clonacion de voz, control expresivo, generación multiespeaker y streaming mediante servidores especializados. Su inferencia oficial utiliza texto como contenido que se quiere pronunciar y audio de referencia para obtener tokens de voz o clonar el timbre:

```text
Texto de respuesta + audio de referencia opcional -> Fish Audio S2 Pro -> audio
```

Se utilizara como backend TTS avanzado dentro de un pipeline, no como agente S2S nativo:

```text
Audio usuario -> STT -> LLM -> texto -> Fish Audio S2 Pro -> audio
```

La distribucion actual `fish-speech 2.0.0` no contiene `fish_speech.models.s2`, `fish_speech.utils.audio` ni la clase `S2Model` utilizada por el prototipo existente. El notebook `speech-to-speech/01_S2S_FishSpeech_S2Pro.ipynb` no debe presentarse como una implementacion funcional hasta sustituir esa API por una ruta oficial compatible.

La documentación de Fish Speech recomienda una GPU con al menos 24 GB para inferencia. Se deben comprobar la memoria real, la version de CUDA, las dependencias y la licencia de pesos, codecs y modelos auxiliares.

#### CSM-1B

CSM-1B genera audio a partir de texto y contexto de audio. Su documentación indica que no es un LLM multimodal general, no genera texto y necesita otro modelo para producir la respuesta textual. Su soporte principal es el inglés.

Se clasificara como generador de voz contextual y no como agente S2S completo:

```text
Audio -> STT o LLM -> texto -> CSM-1B -> audio
```

CSM requiere una GPU compatible con CUDA y el acceso a sus modelos de Hugging Face esta sujeto a condiciones de uso.

## 5. Comparacion para decidir

| Criterio | Realtime comercial | Pipeline local | Qwen2.5-Omni | Moshi | GLM-4-Voice | Fish S2 Pro / CSM |
|---|---|---|---|---|---|---|
| Tipo | Servicio integrado | STT-LLM-TTS | S2S nativo | S2S nativo | S2S nativo | TTS contextual |
| Entrada | Audio | Audio y texto intermedio | Audio y texto | Audio | Audio | Texto y audio de referencia |
| Salida | Audio | Audio | Audio y texto | Audio y texto | Audio y texto | Audio |
| Español | Depende del proveedor | Configurable | Candidato documentado | No prioritario | No documentado oficialmente | Fish sí; CSM limitado |
| Streaming | Si | Requiere implementarlo | Si | Si, full-duplex | Si | Depende del backend |
| Latencia inicial | Baja | Media, optimizable | Potencialmente baja | Muy baja | Baja | Depende del backend |
| Privacidad | Baja o media | Alta | Alta | Alta | Alta | Alta si se ejecuta localmente |
| Facilidad de prototipo | Alta | Alta | Media | Media | Baja | Media |
| Modularidad | Media | Alta | Media | Baja | Baja | Alta dentro del pipeline |
| GPU local | No necesariamente | Recomendable | Necesaria | Aproximadamente 24 GB | CUDA necesaria | Al menos 24 GB recomendados |
| Licencia orientativa | Proveedor | Componentes separados | Apache-2.0 | Codigo MIT/Apache; pesos CC-BY | Codigo Apache-2.0; pesos propios | Licencias propias; revisar codigo y pesos |

## 6. Recomendacion para el proyecto

Se propone crear el nuevo notebook con una implementación S2S principal, una línea base modular y referencias de comparación:

### Implementacion principal: Qwen2.5-Omni open source

Usar `Qwen/Qwen2.5-Omni-3B` mediante Transformers y una inferencia local por turnos con archivos WAV. Esta fase permite comprobar de forma autocontenida:

- Latencia de respuesta.
- Calidad de la conversacion.
- Interrupciones.
- Deteccion de turnos.
- Diferencia frente al pipeline actual sin depender de un servicio de pago.

### Linea base local: pipeline modular

Usar `Pipecat` o `LiveKit Agents` como orquestador y conectar:

```text
Silero VAD -> faster-whisper -> Ollama -> Piper/Kokoro
```

Esta linea base permite medir cuanto se aproxima la experiencia local a la opcion realtime comercial. No debe describirse como S2S nativo, aunque la experiencia final sea de voz a voz.

### Experimento open source S2S: Qwen2.5-Omni

Qwen2.5-Omni-3B se utilizara como primera implementación S2S nativa en local. Si la GPU permite una variante mayor, se probara Qwen2.5-Omni-7B cuantizado. Se compararan latencia, calidad en español, respuesta textual y hablada, estabilidad y consumo con el pipeline modular.

Moshi se mantendra como referencia de S2S full-duplex y baja latencia, pero su evaluación en español quedara condicionada a una validación lingüística inicial. GLM-4-Voice se mantendra como referencia avanzada para chino e ingles.

### TTS avanzado: Fish Audio S2 Pro y CSM-1B

Fish Audio S2 Pro y CSM-1B se evaluaran como backends de síntesis contextual dentro del pipeline modular. No se utilizaran para afirmar que se ha construido un agente S2S nativo.

La recomendacion open source inicial es:

```text
Experimento S2S nativo principal: Qwen2.5-Omni-3B
Experimento S2S nativo ampliado: Qwen2.5-Omni-7B cuantizado
Linea base local: faster-whisper + Ollama + Piper/Kokoro
Referencia S2S full-duplex: Moshi
TTS avanzado: Fish Audio S2 Pro o CSM-1B
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
- Detectar si se ejecuta localmente o en Google Colab.
- Seleccionar `cpu` o `cuda` mediante `torch.cuda.is_available()`.
- Usar `/content` y `files.upload()` en Colab.
- Cargar variables desde `.env`.
- No mostrar ni guardar claves en salidas del notebook.

### Seccion 3: Transporte de audio

- Configurar WebRTC, LiveKit o captura local.
- Probar entrada y salida de audio.
- Medir frecuencia de muestreo y canales.
- Comenzar con archivos WAV antes de incorporar microfono, streaming e interrupciones.

### Seccion 4: Conexion con el modelo

- Inicializar Qwen2.5-Omni como flujo S2S open source principal.
- Mantener una implementacion de referencia con el pipeline local modular.
- Probar Qwen2.5-Omni como flujo S2S nativo principal.
- Mantener Moshi y GLM-4-Voice como referencias experimentales segun soporte linguistico y hardware.
- Mantener Fish Audio S2 Pro y CSM-1B como backends TTS contextuales, no como S2S nativos.
- Configurar instrucciones, voz y modelo.
- Implementar eventos de audio, texto parcial y final de turno.

### Seccion 5: Conversacion e interrupciones

- Detectar cuando el usuario empieza a hablar.
- Cancelar la respuesta de audio en curso.
- Enviar audio por fragmentos.
- Mostrar opcionalmente transcripcion y latencia.

### Seccion 6: Prueba controlada

- Ejecutar tres preguntas en español y repetirlas en los idiomas seleccionados.
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
- Repetir las pruebas en español y, si procede, en ingles y portugues.
- Registrar el modo de idioma del STT: idioma fijado o deteccion automatica.

### Seccion 8: Conclusion

- Comparar realtime comercial, pipeline local, S2S nativo y TTS contextual.
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

Para las pruebas multilingues se debe mantener una configuración explícita de idioma y voz:

```python
LANGUAGE = "es"
LANGUAGE_NAMES = {
    "es": "español",
    "en": "inglés",
    "pt": "portugués",
}
```

Cada idioma requiere un modelo STT y una voz TTS compatibles. No se debe reutilizar una voz española para evaluar inglés o portugués sin documentar esa limitación.

Estas variables son orientativas. El archivo `.env` debe permanecer ignorado por Git.

## 9. Fuentes consultadas

Fuentes oficiales consultadas el 26 de agosto de 2026:

- Qwen2.5-Omni: https://github.com/QwenLM/Qwen2.5-Omni
- Modelo Qwen2.5-Omni-7B: https://huggingface.co/Qwen/Qwen2.5-Omni-7B
- Moshi: https://github.com/kyutai-labs/moshi
- Modelo Moshiko: https://huggingface.co/kyutai/moshiko-pytorch-bf16
- GLM-4-Voice: https://github.com/zai-org/GLM-4-Voice
- CSM: https://github.com/SesameAILabs/csm
- Fish Speech: https://github.com/fishaudio/fish-speech
- Inferencia de Fish Speech: https://speech.fish.audio/inference/

Los repositorios y model cards pueden cambiar. En el experimento se debe registrar la URL, la fecha, el commit o revision, el identificador del modelo y la licencia exacta de cada componente.

## 10. Decision pendiente

Antes de implementar el notebook se deben fijar:

1. Confirmar el primer prototipo S2S nativo con Qwen2.5-Omni-3B en Colab.
2. Confirmar si el pipeline local se ejecutara con microfono local o con LiveKit autohospedado.
3. Probar Qwen2.5-Omni-7B cuantizado si la GPU de Colab tiene memoria suficiente.
4. Comprobar el soporte real de español, streaming e interrupciones en Qwen2.5-Omni.
5. Evaluar Moshi solo como referencia si la limitación al inglés impide una comparación principal en español.
6. Documentar GLM-4-Voice como referencia chino/inglés y no como candidato español sin validación.
7. Mantener Fish Speech S2 Pro y CSM-1B en la categoría TTS contextual.
8. Medir el hardware disponible para cada alternativa local.
9. Fijar las métricas, versiones, revisiones y fecha de consulta.
