# Agente de voz online frente a agente local

## 1. Alcance

Este documento compara dos formas de construir un agente de voz: la version online documentada en `speech-to-text-to-speech/Version_online_agente_voz.ipynb` y una alternativa local.

- **Version online**: usa servicios externos como OpenAI, ElevenLabs y LiveKit Cloud.
- **Version local**: ejecuta el procesamiento en el equipo utilizando modelos y librerias open source.

La comparacion se centra en privacidad, calidad, coste, latencia, mantenimiento y posibilidades de mejora.

## 2. Arquitectura online de Lesson 4

```text
Microfono del navegador
        |
        v
LiveKit / WebRTC
        |
        v
Silero VAD
        |
        v
OpenAI STT
        |
        v
OpenAI GPT-4o
        |
        v
ElevenLabs TTS
        |
        v
LiveKit / navegador
```

| Componente | Servicio o libreria | Funcion |
|---|---|---|
| Transporte | LiveKit Cloud o servidor LiveKit | Lleva el audio entre navegador y agente |
| VAD | Silero | Detecta los turnos de voz |
| STT | OpenAI | Convierte la voz en texto |
| LLM | GPT-4o | Interpreta la pregunta y genera la respuesta |
| TTS | ElevenLabs | Convierte texto en voz |
| Orquestacion | LiveKit Agents | Coordina el pipeline de voz |
| Interfaz | Navegador y panel LiveKit | Captura el microfono y reproduce audio |

## 3. Arquitectura local

```text
Microfono del equipo
        |
        v
sounddevice o LiveKit autohospedado
        |
        v
Silero VAD
        |
        v
faster-whisper
        |
        v
Ollama + modelo local
        |
        v
Piper, Kokoro u otro TTS local
        |
        v
Altavoces o navegador
```

| Componente | Alternativa local | Funcion |
|---|---|---|
| Captura | `sounddevice` | Graba audio del microfono |
| Transporte | Audio local o LiveKit autohospedado | Comunica el cliente y el agente |
| VAD | Silero VAD | Detecta voz y silencio |
| STT | `faster-whisper` o `whisper.cpp` | Transcribe localmente |
| LLM | Ollama, `llama.cpp` o LocalAI | Genera la respuesta |
| TTS | Piper, Kokoro o Coqui TTS | Sintetiza localmente |
| Orquestacion | Python, LiveKit Agents o Pipecat | Coordina la conversacion |

## 4. Que significa Speech-to-Text-to-Speech

Tanto la arquitectura online como la local son sistemas de voz de extremo a extremo. En ambos casos, el usuario habla, el sistema convierte esa voz en texto, genera una respuesta textual y vuelve a convertirla en voz.

La denominacion mas precisa es:

```text
Speech -> Speech-to-Text -> LLM -> Text-to-Speech -> Speech
```

Tambien puede expresarse de forma abreviada como:

```text
Speech-to-Text-to-Speech
```

Aunque el nombre resume el principio, en realidad existe un modelo de lenguaje entre la transcripcion y la sintesis. Por eso, la cadena completa tiene tres etapas principales:

1. **Speech-to-Text (STT)**: convierte la voz del usuario en texto.
2. **LLM**: interpreta el texto y genera una respuesta.
3. **Text-to-Speech (TTS)**: convierte la respuesta textual en voz.

La diferencia entre las dos versiones no esta en el flujo, sino en donde se ejecuta cada etapa:

| Etapa | Version online | Version local |
|---|---|---|
| Entrada | Audio del usuario | Audio del usuario |
| STT | Servicio online de transcripcion | Modelo local como `faster-whisper` |
| Generacion | Modelo online como GPT-4o | Modelo local ejecutado con Ollama |
| TTS | Servicio online como ElevenLabs | Motor local como Piper o Kokoro |
| Salida | Audio reproducido al usuario | Audio reproducido al usuario |

### Flujo online

```text
Voz del usuario
        -> STT online
        -> LLM online
        -> TTS online
        -> Voz de respuesta
```

### Flujo local

```text
Voz del usuario
        -> STT local
        -> LLM local
        -> TTS local
        -> Voz de respuesta
```

En consecuencia, ambos sistemas son Speech-to-Text-to-Speech. La version online prioriza normalmente calidad, latencia y facilidad de despliegue; la version local prioriza privacidad, control de los modelos y ausencia de coste por peticion.

## 5. Ventajas de la version online

### Calidad

Los servicios online suelen ofrecer modelos grandes con mejor capacidad de razonamiento, transcripcion y sintesis de voz. GPT-4o y ElevenLabs normalmente producen respuestas y voces más naturales que una configuracion local pequena.

### Latencia

El procesamiento se realiza en servidores especializados. En un equipo sin GPU, la version online puede responder antes que una cadena local basada en CPU.

### Experiencia de voz

LiveKit Agents ya proporciona abstracciones para streaming de audio, interrupciones, deteccion de turnos, herramientas y sesiones de agente. Esto aproxima la experiencia a un producto de voz en tiempo real.

### Mantenimiento

No es necesario descargar, actualizar y optimizar todos los modelos. El proveedor mantiene la infraestructura de inferencia.

### Escalabilidad

Es más sencillo atender a varios usuarios o desplegar el agente en diferentes dispositivos, aunque aumenta el coste de infraestructura y consumo.

## 6. Inconvenientes de la version online

### Privacidad

El audio, las transcripciones y posiblemente el contexto de la conversacion salen del equipo y se procesan en servidores externos. Hay que revisar politicas de retencion, tratamiento de datos y cumplimiento legal.

### Coste variable

Normalmente se paga por consumo. El coste puede depender de:

- Minutos de audio transcrito.
- Tokens de entrada y salida del LLM.
- Caracteres o segundos de audio sintetizado.
- Numero de conexiones o minutos de LiveKit.
- Almacenamiento, observabilidad y despliegue.

Los precios cambian con frecuencia y dependen del modelo y del plan. Deben consultarse en las paginas oficiales antes de presupuestar el proyecto. No conviene presentar un precio fijo en la memoria sin indicar la fecha de consulta.

### Dependencia de proveedores

El agente depende de la disponibilidad, limites de uso, cambios de API y condiciones comerciales de varios proveedores. Un cambio de version puede obligar a actualizar plugins o codigo de integracion.

### Claves y configuracion

La version online necesita variables como:

```text
OPENAI_API_KEY
ELEVEN_API_KEY
LIVEKIT_API_KEY
LIVEKIT_API_SECRET
LIVEKIT_URL
```

Estas claves deben mantenerse fuera del notebook y no deben subirse a Git. Una clave expuesta puede generar consumo no autorizado.

### Limitaciones operativas

Pueden aparecer:

- Limites de peticiones.
- Errores de autenticacion.
- Timeouts.
- Caidas del servicio.
- Diferencias entre regiones.
- Cambios en modelos retirados o renombrados.
- Costes inesperados por reintentos o conversaciones largas.

### Problemas de librerias

La integracion depende de versiones compatibles entre:

- `livekit-agents`.
- Plugins de LiveKit.
- SDK de OpenAI.
- SDK de ElevenLabs.
- Python.
- Jupyter y sus extensiones.

Un notebook puede funcionar con una combinacion concreta y dejar de hacerlo despues de una actualizacion si no se fijan versiones.

## 7. Ventajas de la version local

### Privacidad

El audio, las transcripciones y las respuestas pueden permanecer en el equipo. Esto es especialmente importante para datos personales, conversaciones sensibles o material de investigacion.

### Coste por uso

Una vez descargados los modelos, no existe un coste por cada peticion. El coste real es el equipo, la electricidad, el almacenamiento y el tiempo de mantenimiento.

### Control

Se puede elegir el modelo, la cuantizacion, el idioma, el prompt, el historial y el modo de procesamiento. Tambien se puede inspeccionar cada paso del pipeline.

### Reproducibilidad

El entorno puede fijarse con versiones concretas y los mismos modelos pueden utilizarse para repetir experimentos.

### Independencia

El agente puede continuar funcionando sin Internet y sin depender de la disponibilidad de una API externa.

## 8. Inconvenientes de la version local

### Calidad dependiente del hardware

La calidad y velocidad dependen de CPU, GPU, memoria y almacenamiento. Un modelo pequeno como `qwen2.5:3b` puede ser suficiente para dialogos simples, pero no iguala necesariamente a GPT-4o en razonamiento o seguimiento de instrucciones.

### Latencia

En CPU hay que procesar de forma local la transcripcion, el LLM y la sintesis. La primera carga de cada modelo puede ser lenta y consumir bastante memoria.

### Instalacion

Es necesario instalar paquetes, descargar pesos, configurar modelos y comprobar compatibilidades. Algunos modelos requieren CUDA, ONNX Runtime, PyTorch u otras dependencias nativas.

### Calidad de voz

Los motores TTS locales pueden tener menos voces, menor expresividad o peor prosodia que ElevenLabs. La calidad depende de la voz concreta y de sus pesos.

### Mantenimiento

El propietario del proyecto debe actualizar las librerias, revisar licencias, corregir errores de dispositivos de audio y controlar el consumo de recursos.

### Escalabilidad

Un unico equipo local no es una solucion sencilla para muchos usuarios simultaneos. Para escalar se necesita mas hardware o una infraestructura propia.

## 9. Librerias y frameworks open source recomendables

### 8.1 STT: `faster-whisper`

Es la opcion recomendada para mantener el proyecto en Python. Usa CTranslate2, puede ejecutarse en CPU o GPU y permite utilizar el filtro VAD integrado.

Configuracion inicial:

```python
from faster_whisper import WhisperModel

model = WhisperModel(
    "base",
    device="cpu",
    compute_type="int8",
)
```

Si la calidad no es suficiente, se puede probar `small`. Si se dispone de GPU, se puede valorar `medium` o `large-v3`.

Alternativas:

- `whisper.cpp`: ejecutable y motor ligero, interesante para CPU.
- Vosk: bajo consumo y funcionamiento offline, pero normalmente con menor calidad en lenguaje libre.
- WhisperX: incorpora alineacion temporal y diarizacion, pero aumenta la complejidad.

### 8.2 VAD: Silero VAD

Es una eleccion adecuada para detectar voz en tiempo real. La mejora necesaria no es cambiarlo, sino conectarlo al flujo de `sounddevice` para cerrar automaticamente cada turno tras un periodo de silencio.

### 8.3 LLM: Ollama

Ollama es apropiado para este proyecto porque simplifica la descarga y ejecucion de modelos y ofrece una API local.

Alternativas:

- `llama.cpp`: control directo y buen rendimiento con modelos GGUF.
- LocalAI: API compatible con OpenAI para ejecutar distintos modelos locales.
- vLLM: muy potente para servidores con GPU y varias peticiones, menos adecuado para un equipo personal modesto.
- Text Generation Inference: orientado a despliegues de modelos en servidor.

### 8.4 TTS: Piper y alternativas

Piper es ligero y funciona bien en una demo local. Sin embargo, el repositorio original de Rhasspy Piper fue archivado y remite a un proyecto posterior. Hay que comprobar el repositorio activo, la licencia del motor y la licencia de cada voz antes de distribuirlo.

Alternativas que merece la pena evaluar:

- Kokoro TTS: buena relacion entre calidad y recursos, revisando disponibilidad de modelos y licencias.
- Coqui TTS o XTTS: mas posibilidades de voz, con mayor consumo y requisitos de licencia.
- Piper mantenido por el proyecto sucesor: opcion ligera si la compatibilidad con Windows es adecuada.

### 8.5 Orquestacion de voz

#### Python propio

Es la opcion más sencilla para el TFM. Cada componente se puede probar y medir de forma independiente:

```text
sounddevice -> VAD -> STT -> LLM -> TTS -> sounddevice
```

#### LiveKit Agents

Es la alternativa más parecida a `Version_online_agente_voz`. El framework es open source y proporciona sesiones, streaming, turn detection, herramientas e integracion con proveedores. Puede utilizar LiveKit Cloud o un servidor autohospedado.

El reto es crear adaptadores compatibles con las interfaces de STT, LLM y TTS para los componentes locales.

#### Pipecat

Framework open source orientado a pipelines de voz y tiempo real. Puede ser interesante si se desea conectar componentes de audio, STT, LLM y TTS con una arquitectura de pipeline explícita.

#### LangChain o LlamaIndex

Son útiles para herramientas, RAG, memoria y conexiones con documentos. No sustituyen por sí mismos al transporte de audio ni al STT/TTS. Se incorporarían como capa de razonamiento o conocimiento dentro del agente.

#### Rasa

Puede ser útil para dialogos basados en intenciones y flujos controlados. Es menos directo si el objetivo principal es un asistente generativo basado en Ollama.

## 10. Mejoras locales posibles

### Mejora 1: VAD automatico

Sustituir el segundo clic del boton por deteccion de silencio con Silero VAD. El usuario pulsa una vez para iniciar la escucha y el agente cierra el turno automaticamente.

### Mejora 2: streaming

Procesar la transcripcion y la respuesta por fragmentos cuando sea posible. Esto reduce el tiempo percibido, aunque aumenta la complejidad de sincronizar audio y texto.

### Mejora 3: modelo LLM

Comparar al menos dos modelos mediante un conjunto fijo de preguntas:

- `qwen2.5:3b` para bajo consumo.
- Un modelo mayor compatible con el equipo para mejorar razonamiento.

Deben medirse calidad, latencia, memoria y tasa de respuestas incompletas.

### Mejora 4: respuestas orientadas a voz

Usar un prompt que limite la longitud y evite formatos poco naturales al hablar:

```text
Responde en español, con frases cortas y naturales. No uses tablas ni listas largas.
```

### Mejora 5: herramientas locales

El agente puede incorporar funciones controladas para:

- Consultar archivos locales.
- Buscar en una base de datos.
- Ejecutar calculos.
- Consultar un calendario local.
- Recuperar documentos mediante RAG.

Las herramientas deben validar entradas y limitar las acciones que pueden ejecutar.

### Mejora 6: memoria y RAG

Para responder sobre documentos del TFM se puede añadir:

```text
documentos -> embeddings locales -> base vectorial -> contexto -> Ollama
```

Opciones habituales son Chroma, FAISS o una base SQL con extension vectorial. La eleccion depende del tamaño y del tipo de consulta.

### Mejora 7: observabilidad

Registrar por turno:

- Duracion del audio.
- Tiempo de STT.
- Tiempo de LLM.
- Tiempo de TTS.
- Longitud de la respuesta.
- Errores por componente.
- Consumo de memoria.

Esto permite justificar tecnicamente la comparacion con la version online.

### Mejora 8: gestion de errores

Añadir timeouts, reintentos limitados, cancelacion de audio y limpieza de archivos temporales. Nunca conviene reintentar indefinidamente una peticion a Ollama o dejar abierto un stream de audio.

## 11. Problemas concretos de la version online

| Problema | Causa habitual | Prevencion |
|---|---|---|
| `401 Unauthorized` | Clave ausente, incorrecta o caducada | Validar `.env` y no exponer claves |
| `429 Too Many Requests` | Limite de peticiones o cuota | Controlar ritmo, reintentos y presupuesto |
| `Timeout` | Red lenta, modelo ocupado o audio largo | Configurar timeout y limitar duracion |
| Error de plugin | Versiones incompatibles | Fijar versiones en `requirements.txt` |
| Modelo no disponible | Cambio o retirada del modelo | Registrar modelo y fecha de prueba |
| Coste inesperado | Respuestas largas o reintentos | Limitar tokens y monitorizar consumo |
| Fallo de LiveKit | Token, URL o sala incorrecta | Comprobar credenciales y ciclo de vida |
| Problemas de navegador | Permisos de microfono o autoplay | Gestionar permisos e iframe correctamente |

## 12. Problemas concretos de la version local

| Problema | Causa habitual | Prevencion |
|---|---|---|
| Transcripcion lenta | Modelo grande o CPU limitada | Usar `base`, `small` o cuantizacion |
| Audio sin señal | Dispositivo incorrecto | Enumerar dispositivos y seleccionar indice |
| Respuesta incompleta | `num_predict` bajo o interrupcion | Ajustar limite y controlar cancelacion |
| Voz artificial | TTS o voz limitada | Comparar Piper, Kokoro y otras voces |
| Falta de memoria | Modelos cargados simultaneamente | Liberar modelos y medir RAM |
| Instalacion rota | Dependencias nativas incompatibles | Fijar versiones y usar entorno virtual |
| Modelo no descargado | Primera ejecucion o cache incompleta | Verificar rutas y descargar previamente |
| Licencia incompatible | Pesos o voces con condiciones distintas | Registrar licencias de cada componente |

## 13. Recomendacion para este proyecto

La mejor estrategia es conservar dos implementaciones comparables:

### Prototipo online

```text
LiveKit + Silero VAD + OpenAI STT + GPT-4o + ElevenLabs
```

Sirve como referencia de calidad, latencia y experiencia de usuario.

### Prototipo local

```text
sounddevice + Silero VAD + faster-whisper + Ollama + Piper/Kokoro
```

Sirve para estudiar privacidad, coste, consumo de recursos y reproducibilidad.

El desarrollo local recomendado es:

1. Mantener la prueba actual con botón manual.
2. Integrar Silero VAD para detectar automaticamente el final del turno.
3. Centralizar el historial y las métricas dentro de `VoiceAgent`.
4. Comparar varios modelos LLM y TTS.
5. Añadir herramientas o RAG si el agente necesita consultar información del TFM.
6. Integrar LiveKit autohospedado o Pipecat solo cuando el pipeline local sea estable.

## 14. Conclusión

La version online ofrece una experiencia más inmediata y normalmente una calidad superior, pero introduce costes variables, dependencia de Internet, exposición de datos y problemas de compatibilidad entre servicios.

La version local requiere más trabajo de instalacion y optimizacion, pero permite mantener los datos en el equipo, controlar el pipeline y realizar experimentos reproducibles sin coste por peticion.

Para una evaluacion rigurosa conviene comparar ambas con el mismo conjunto de frases y registrar:

```text
calidad de transcripcion
calidad de respuesta
naturalidad de voz
latencia total
consumo de memoria
coste por interaccion
privacidad
robustez ante errores
```

La conclusion no tiene que ser que una version es universalmente mejor. La eleccion depende del objetivo: la version online prioriza calidad y facilidad de despliegue; la local prioriza control, privacidad y ausencia de coste variable.
