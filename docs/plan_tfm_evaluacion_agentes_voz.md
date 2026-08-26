# Plan de TFM: comparativa de arquitecturas de voz abiertas

## 1. Idea principal

El TFM estudiara y comparara tres arquitecturas para crear agentes de voz conversacionales:

1. Un pipeline `Speech-to-Text-to-Speech` compuesto por STT, LLM y TTS independientes.
2. Una variante del pipeline anterior con un modelo Transformer cargado directamente desde Hugging Face.
3. Un agente `Speech-to-Speech` basado en un modelo abierto que reciba y genere audio.

La comparativa se realizara utilizando modelos y herramientas open source o con pesos disponibles para investigacion. El objetivo sera analizar que arquitectura ofrece el mejor equilibrio entre calidad, latencia, estabilidad, privacidad, consumo de recursos y facilidad de integracion.

Ya se ha probado una primera version local del pipeline `Speech-to-Text-to-Speech` con `faster-whisper`, Ollama y Piper. Esta implementacion servira como linea base del estudio. El siguiente paso sera seleccionar e integrar candidatos de audio conversacional bajo condiciones comparables.

El estudio distinguira dos entornos de ejecucion. La linea base con Ollama se mantendra en el ordenador local para medir una configuracion reproducible y privada. Los modelos S2S y los modelos Transformer de mayor tamaño se evaluaran principalmente en Google Colab con CUDA, registrando en cada ejecución el tipo de GPU, la memoria disponible, la versión de CUDA, la versión de PyTorch y el tiempo de carga. Los resultados de hardware diferente no se presentaran como una comparación pura de modelos sin declarar esta diferencia.

Como nueva arquitectura experimental se establece un pipeline modular ejecutado mediante un orquestador sencillo en Python:

```text
Whisper / faster-whisper (STT)
        -> modelo Transformer instruct (LLM)
        -> F5-TTS o Fish Speech (TTS)
```

Esta arquitectura se evaluara inicialmente por turnos. El audio se convertira en texto, el LLM generara una respuesta y el TTS producira el audio de salida. La modularidad permite sustituir cada etapa y medir sus latencias y errores de forma independiente. F5-TTS y Fish Speech seran alternativas de TTS, no componentes simultaneos del mismo pipeline.

## 2. Arquitecturas objeto de comparacion

### 2.1 Pipeline modular con Ollama

Esta es la arquitectura local que ya se ha probado en `speech-to-text-to-speech/Ollama_Local_Agent.ipynb`:

```text
Microfono -> Silero VAD -> faster-whisper -> Ollama -> Piper/Kokoro -> altavoces
```

El orquestador Python envia los mensajes a la API local de Ollama, normalmente en `http://127.0.0.1:11434`. Ollama se encarga de servir el modelo y abstrae parte de la carga, la gestion de memoria y la comunicacion con el LLM.

Componentes:

| Etapa | Componente | Funcion |
|---|---|---|
| Deteccion de voz | Silero VAD | Separa voz y silencio |
| STT | `faster-whisper` | Convierte audio en texto |
| LLM | Ollama + Qwen, Llama o Mistral | Genera la respuesta mediante API HTTP |
| TTS | Piper o Kokoro | Convierte la respuesta en audio |
| Orquestacion | Python | Conserva el historial y coordina el turno |

Ventajas principales: instalacion sencilla, proceso del modelo separado, cambio rapido de modelos y API estable. Inconvenientes: menor control directo sobre PyTorch, dependencia del servicio local de Ollama y menor visibilidad sobre algunos detalles de la inferencia.

Cada etapa puede sustituirse de forma independiente y permite medir por separado la transcripcion, la generacion y la sintesis.

### 2.2 Pipeline modular con Transformers

Esta es la arquitectura experimental que se implementara en `speech-to-text-to-speech/Transformers_Local_Voice_Agent.ipynb`:

```text
Microfono -> Whisper / faster-whisper -> modelo Transformer instruct -> F5-TTS o Fish Speech -> altavoces
```

El orquestador sera Python y conectara las etapas por turnos. El STT convertira el audio en texto, el modelo Transformer generara la respuesta y el TTS la convertira de nuevo en audio. Como primera prueba se puede utilizar Piper, ya disponible en el proyecto, para validar el LLM sin añadir la complejidad de F5-TTS o Fish Speech.

Los candidatos para el LLM son `Qwen/Qwen2.5-7B-Instruct`, `meta-llama/Llama-3.1-8B-Instruct`, `google/gemma-2-9b-it` y `mistralai/Mistral-7B-Instruct-v0.3`. Se comenzara con `Qwen/Qwen2.5-3B-Instruct` si la memoria del equipo no permite ejecutar los modelos mayores.

Los candidatos para TTS son F5-TTS y Fish Speech. Se evaluaran como backends alternativos, no simultaneamente. Piper se conservara como linea base practica porque ya existe una voz española local en `models/piper/`.

Esta arquitectura permite controlar directamente el tokenizer, el dispositivo, la cuantizacion, los parametros de generacion y las metricas de PyTorch. Sin embargo, Transformers y Ollama son runtimes diferentes: para atribuir una diferencia al runtime hay que utilizar el mismo modelo, cuantizacion, prompt, hardware y parametros de generacion.

La primera fase sera por turnos y con archivos o buffers WAV. El streaming, el VAD en tiempo real y las interrupciones se incorporaran despues de validar cada componente por separado.

### 2.3 Candidatos de audio conversacional

Esta categoria se investigara para determinar si el modelo recibe audio y genera audio directamente, o si en realidad necesita texto como entrada. El criterio debe comprobarse en la version exacta utilizada, porque un modelo TTS o text-to-audio no es automaticamente un modelo Speech-to-Speech nativo.

```text
Microfono -> transporte o interfaz -> modelo de audio -> altavoces
```

Los candidatos iniciales son Qwen2.5-Omni, Moshi y GLM-4-Voice, junto con Fish Audio S2 Pro y CSM-1B como referencias TTS contextuales. Dia se mantendra como candidato exploratorio hasta verificar su formato de entrada, salida, soporte de español y funcionamiento en streaming.

No se afirmara a priori que estos sean los unicos modelos S2S open source ni que ambos realicen conversión directa audio-audio. El plan registrara para cada candidato si cumple realmente estas condiciones:

| Criterio | Comprobacion |
|---|---|
| Entrada de audio | Acepta voz del usuario sin transcripcion externa |
| Salida de audio | Genera voz como respuesta sin TTS externo |
| Conversacion | Mantiene turnos y contexto conversacional |
| Español | Tiene soporte documentado y resultados aceptables |
| Tiempo real | Permite streaming, baja latencia o generacion incremental |
| Licencia | Codigo, pesos, codecs y auxiliares son compatibles con el estudio |

Si un modelo falla la comprobacion de entrada audio-audio, se clasificara como TTS o text-to-audio y se incorporara a la comparacion modular, no a la categoria S2S nativa.

## 3. Objetivo general

Diseñar y aplicar una metodologia reproducible para comparar una linea base local con Ollama, una variante modular con Transformers y TTS alternativos, y candidatos de audio conversacional basados en modelos abiertos, en un escenario comun de conversacion en espanol.

## 4. Objetivos especificos

1. Revisar las plataformas comerciales realtime como contexto del estado del arte.
2. Seleccionar modelos abiertos para las arquitecturas STT-LLM-TTS, Transformers y audio conversacional.
3. Definir una arquitectura y un conjunto de tareas iguales para las tres alternativas.
4. Implementar un cliente o adaptador comun para cada arquitectura y backend.
5. Medir objetivamente latencia, calidad, estabilidad y consumo.
6. Evaluar subjetivamente la naturalidad y la experiencia conversacional.
7. Analizar privacidad, licencias y requisitos de despliegue.
8. Proponer una recomendacion basada en los datos obtenidos.

## 5. Preguntas de investigacion

### Pregunta principal

Que arquitectura abierta ofrece el mejor equilibrio entre calidad conversacional, latencia, privacidad, consumo de recursos y facilidad de integracion para un agente de voz en espanol?

### Preguntas secundarias

- Que diferencias existen entre los candidatos de audio conversacional y los pipelines STT-LLM-TTS?
- Que arquitectura responde antes despues de que el usuario termina de hablar?
- Como gestionan las diferentes arquitecturas las interrupciones y los cambios de turno?
- Que impacto tiene la calidad de STT en la respuesta final del agente?
- Que coste de hardware y mantenimiento requiere cada pipeline open source?
- Que arquitectura ofrece mejores mecanismos de observabilidad, control y recuperacion ante errores?
- Hasta que punto un modelo S2S abierto puede ofrecer resultados comparables al pipeline modular?

## 6. Modelos y tecnologias candidatas

La seleccion definitiva debe hacerse antes de implementar la fase experimental y debe registrarse con la fecha de consulta, version, licencia, requisitos de hardware y condiciones de uso de cada modelo.

### Tecnologias comerciales como contexto

- OpenAI Realtime API.
- Gemini Live API.
- ElevenLabs Conversational AI.

No seran el objeto principal de la evaluacion por sus requisitos de pago. Se documentaran para contextualizar el estado del arte y comparar las capacidades que se pueden conseguir con soluciones abiertas.

### Modelos abiertos para el pipeline

### Linea base open source

```text
Silero VAD -> faster-whisper -> Ollama -> Piper o Kokoro
```

Esta linea base no debe presentarse como Speech-to-Speech nativo. Su funcion sera servir como referencia de coste, privacidad, control y rendimiento local.

### Posibilidades de pipeline STT-LLM-TTS ordenadas por eficiencia

En esta tabla, eficiencia significa principalmente menor latencia y menor consumo de CPU, RAM y GPU, manteniendo una calidad razonable en español. Las configuraciones no son S2S nativas; se incluyen como líneas base y alternativas modulares.

| Nivel | STT | LLM | TTS | Hardware orientativo | Eficiencia | Calidad esperada | Uso recomendado |
|---|---|---|---|---|---|---|---|
| 1. Muy ligero | `faster-whisper tiny` | `llama3.2:1b` con Ollama | Piper | CPU y poca RAM | Muy alta | Baja-media | Prueba mínima y equipos limitados |
| 2. Ligero | `faster-whisper base` | Qwen2.5 1.5B | Piper | CPU | Alta | Media | Prototipo local sencillo |
| 3. Equilibrado local | `faster-whisper base` | `qwen2.5:3b` con Ollama | Piper | CPU y 8-16 GB de RAM | Media-alta | Media-alta | Opcion recomendada para el ordenador actual |
| 4. Mejor reconocimiento | `faster-whisper small` | `qwen2.5:3b` con Ollama | Piper | CPU potente o GPU | Media | Alta en reconocimiento | Español, ruido y frases complejas |
| 5. Mejor LLM local | `faster-whisper base` | Qwen2.5-7B-Instruct | Piper | GPU de 16-24 GB | Media | Alta | Comparacion academica de modelos |
| 6. GPU equilibrada | `faster-whisper small` en CUDA | Qwen2.5-7B-Instruct cuantizado | Piper | T4, L4 o superior | Alta en GPU | Alta | Experimento principal en Colab |
| 7. TTS mas natural | `faster-whisper small` | Qwen 3B o 7B | Fish Audio S2 Pro | GPU de al menos 24 GB | Media | Muy alta en voz | Comparar Piper con TTS avanzado |
| 8. Pipeline realtime | STT por streaming | LLM por streaming | TTS por streaming | GPU y orquestador realtime | Alta en interacción | Alta | Latencia, turnos e interrupciones |

La configuración recomendada para la línea base es:

```text
faster-whisper base -> Ollama qwen2.5:3b -> Piper
```

La configuración recomendada para Colab es:

```text
faster-whisper small -> Qwen2.5-7B-Instruct cuantizado -> Piper
```

Fish Audio S2 Pro se comparara como backend TTS avanzado, no como modelo S2S nativo.

### Modelos para comparar Ollama y Transformers

La comparacion se realizara en dos niveles:

1. **Comparacion del runtime**: mismo modelo, por ejemplo Qwen2.5 3B, ejecutado mediante Ollama y mediante `transformers`.
2. **Comparacion de modelos**: diferentes familias o tamaños ejecutados con el mismo runtime, registrando por separado el efecto del modelo.

Los candidatos iniciales son:

| Modelo | Ollama | Transformers | Observacion |
|---|---|---|---|
| Qwen2.5 3B Instruct | `qwen2.5:3b` | `Qwen/Qwen2.5-3B-Instruct` | Primera prueba y comparación más asequible |
| Qwen2.5 7B Instruct | Etiqueta compatible disponible | `Qwen/Qwen2.5-7B-Instruct` | Candidato multilingue; requiere más memoria |
| Llama 3.1 8B Instruct | Etiqueta compatible disponible | `meta-llama/Llama-3.1-8B-Instruct` | Referencia habitual en investigación |
| Gemma 2 9B Instruct | Etiqueta compatible disponible | `google/gemma-2-9b-it` | Candidato de mayor tamaño |
| Mistral 7B Instruct | Etiqueta compatible disponible | `mistralai/Mistral-7B-Instruct-v0.3` | Referencia ligera y conocida |

Las etiquetas de Ollama dependen de los modelos publicados y de la version instalada. Se anotaran el nombre exacto, la cuantizacion y la revision utilizada en cada ejecucion.

### Modelos abiertos para Speech-to-Speech

| Opcion | Entrada y salida | Idiomas documentados | Tiempo real | GPU orientativa | Papel en el TFM |
|---|---|---|---|---|---|
| Qwen2.5-Omni-3B | Audio, texto, imagen o video -> texto y audio | Multilingue, incluido español | Si, streaming | Al menos 18-22 GB teoricos en BF16 | Primera opcion S2S en Colab |
| Qwen2.5-Omni-7B GPTQ/AWQ | Audio y texto -> texto y audio | Multilingue, incluido español | Si, streaming | Aproximadamente 12-18 GB teoricos según variante | Ampliacion para GPU limitada |
| Qwen2.5-Omni-7B BF16 | Audio y texto -> texto y audio | Multilingue, incluido español | Si, streaming | Aproximadamente 31 GB teoricos; mas en la practica | No adecuada para T4 de 16 GB |
| Moshi / Moshiko | Audio del usuario -> audio del agente y texto auxiliar | Principalmente ingles | Si, full-duplex | Aproximadamente 24 GB | Referencia S2S y de latencia |
| GLM-4-Voice-9B | Voz -> voz y texto | Chino e ingles | Si | CUDA; existe modo int4 | Referencia avanzada, no principal para español |
| CSM-1B | Texto y contexto de audio -> audio | Principalmente ingles | Limitado | GPU CUDA | Generador contextual, no S2S completo |
| Fish Audio S2 Pro | Texto y audio de referencia -> audio | Multilingue, incluido español | Depende del servidor | Al menos 24 GB recomendados | TTS avanzado, no S2S conversacional |
| Dia | Debe verificarse la entrada y salida exactas | Debe verificarse | Debe verificarse | Debe verificarse | Candidato exploratorio pendiente de validación |

Qwen2.5-Omni sera el candidato S2S nativo principal para español. Moshi y GLM-4-Voice se estudiaran como referencias, condicionadas por sus limitaciones lingüísticas. CSM y Fish Audio S2 Pro se evaluaran como TTS contextual, no como S2S nativo. Dia se mantendra únicamente como candidato exploratorio hasta verificar su entrada, salida, idiomas, streaming y licencia.

La seleccion final debe considerar soporte para espanol, entrada y salida de audio, streaming, interrupciones, consumo de GPU, disponibilidad de pesos y licencias del codigo, los pesos, el codec y los modelos auxiliares.

La comparación principal se realizará en español. Se podrán añadir inglés y portugués como idiomas secundarios, siempre que exista un STT y un TTS compatibles y que el modelo S2S declare soporte suficiente. Los resultados de idiomas secundarios se analizarán por separado y no se mezclarán en una única media con los resultados en español.

### Frameworks y herramientas de orquestacion

Los frameworks no son modelos S2S. Proporcionan transporte, captura, streaming, turn-taking, eventos y adaptadores para conectar modelos.

| Framework o herramienta | Funcion en el TFM | Uso recomendado |
|---|---|---|
| **Pipecat** | Orquestacion de pipelines de voz en tiempo real | Comparar pipelines STT-LLM-TTS y conectar servicios o modelos locales |
| **LiveKit Agents** | Transporte WebRTC, sesiones, turnos e interrupciones | Agente interactivo con navegador o servidor LiveKit |
| **Hugging Face Transformers** | Carga y ejecucion directa de modelos compatibles | Experimento academico y control de tokenizer, dispositivo y generacion |
| **Gradio** | Interfaz web para microfono, audio de salida y pruebas | Demo en Colab o Hugging Face Spaces |
| **Python `asyncio`** | Orquestacion minima y control explicito del flujo | Prototipo reproducible por turnos sin framework realtime |
| **SGLang o vLLM** | Servir modelos compatibles con GPU y streaming | Fase posterior si el candidato de audio admite ese runtime |

La primera implementacion utilizara Python y, cuando se ejecute online, Gradio para la interfaz. Pipecat o LiveKit Agents se evaluaran en una fase posterior si se necesita streaming, interrupciones y transporte realtime.

## 7. Escenario comun de evaluacion

Para que la comparacion sea valida, todos los agentes deben resolver el mismo conjunto de tareas.

### Perfil del agente

- Idioma principal: espanol.
- Personalidad e instrucciones identicas o equivalentes.
- Respuestas concisas, de una a tres frases cuando la tarea no requiera mas detalle.
- Sin herramientas externas en la primera fase.
- Sin memoria entre conversaciones, salvo que la prueba mida memoria de forma especifica.
- Misma frase de bienvenida y mismo mensaje de sistema siempre que la plataforma lo permita.

### Conjunto de pruebas

1. Preguntas sencillas de informacion general.
2. Preguntas con nombres propios y numeros.
3. Peticiones con ruido de fondo.
4. Cambios de idioma breves.
5. Frases incompletas y autocorrecciones.
6. Interrupciones mientras habla el agente.
7. Silencios prolongados.
8. Dos turnos consecutivos sobre el mismo tema.
9. Peticiones ambiguas que requieran aclaracion.
10. Conversaciones de duracion corta, media y larga.

Cada prueba debe ejecutarse varias veces y con las mismas condiciones de red y hardware cuando sea posible.

## 8. Metricas objetivas

### 8.1 Latencia

Registrar al menos:

- **Tiempo hasta el primer audio**: desde el final de la intervencion del usuario hasta el primer fragmento de respuesta audible.
- **Tiempo hasta la respuesta completa**: desde el final de la intervencion hasta que termina el audio.
- **Duracion de la respuesta**.
- **Latencia de transcripcion parcial**, si la API la proporciona.
- **Latencia de deteccion del turno**.
- **Tiempo de recuperacion despues de una interrupcion**.

Se deben guardar media, mediana, percentil 95 y desviacion estandar. La mediana representa mejor una conversacion habitual y el percentil 95 permite observar casos lentos.

### 8.2 Calidad del reconocimiento de voz

- Word Error Rate (WER), si se dispone de una transcripcion de referencia.
- Character Error Rate (CER) para nombres y palabras dificiles.
- Porcentaje de numeros, nombres y comandos transcritos correctamente.
- Errores producidos por ruido, acento o pausas.

La transcripcion de referencia debe revisarse manualmente y conservarse junto con el identificador de cada prueba.

### 8.3 Calidad de la respuesta

- Correccion de la respuesta.
- Relevancia.
- Cumplimiento de instrucciones.
- Capacidad de pedir aclaraciones.
- Consistencia entre ejecuciones.
- Tasa de respuestas incompletas o fuera de contexto.
- Tasa de errores de API y respuestas fallidas.

Se puede utilizar una rubrica de 1 a 5 puntos por criterio. Cuando se utilice evaluacion automatica con un LLM, debe complementarse con una muestra evaluada por personas y documentarse el posible sesgo.

### 8.4 Calidad de voz y conversacion

Evaluar mediante una encuesta controlada:

- Naturalidad de la voz.
- Claridad.
- Pronunciacion en espanol.
- Adecuacion de la entonacion.
- Gestion de pausas.
- Naturalidad de los turnos.
- Comodidad de las interrupciones.
- Sensacion general de conversacion.

Se puede utilizar una escala Likert de 1 a 5 y calcular media, mediana e intervalo de confianza cuando el numero de participantes lo permita.

### 8.5 Coste y recursos

Para los pipelines open source se registrara:

- Memoria RAM y memoria de GPU utilizadas.
- Tiempo de CPU y GPU por conversacion.
- Requisitos de hardware para ejecutar cada modelo.
- Coste estimado del hardware, electricidad y mantenimiento.
- Tiempo de instalacion y configuracion.

Para las plataformas comerciales se documentara solo de forma teorica el coste por minuto, tokens, audio o sesiones, segun corresponda. Los precios se anotaran con moneda, plan, fecha de consulta y URL de la documentacion oficial. No se utilizaran para construir los resultados experimentales principales.

### 8.6 Operacion y privacidad

- Tiempo de configuracion.
- Complejidad de la integracion.
- Calidad de la documentacion.
- Disponibilidad de SDK y plugins.
- Observabilidad y logs.
- Gestion de errores y reconexion.
- Limites de uso.
- Dependencia del proveedor.
- Lugar de procesamiento del audio.
- Politicas de retencion y tratamiento de datos.
- Posibilidad de eliminar o exportar datos.

## 8.7 Agregacion de resultados

La unidad basica de análisis será una ejecución completa de un escenario y una configuración. Cada configuración se ejecutará una vez como calentamiento y, después, al menos cinco veces válidas por escenario. Las ejecuciones de calentamiento no entrarán en los resultados.

Para cada métrica numérica se registrarán:

- Número de ejecuciones válidas.
- Media aritmética.
- Mediana.
- Desviación estándar.
- Percentil 95.
- Mínimo y máximo como información complementaria.

La media se calculará sobre las ejecuciones válidas de cada combinación de arquitectura, modelo, idioma, tarea y entorno. No se mezclaran en una misma media resultados de local y Colab, GPU diferentes, idiomas diferentes ni configuraciones con distinto número de repeticiones. Para muestras subjetivas se informará también el número de evaluadores, la media, la mediana y la dispersión de las puntuaciones Likert.

Las latencias se medirán en segundos con `time.perf_counter_ns()` y se definirán mediante eventos comunes. Como mínimo se guardarán:

```text
fin_intervencion_usuario
inicio_primer_audio
fin_audio_respuesta
```

De estos eventos se obtendrán:

```text
TTFA = inicio_primer_audio - fin_intervencion_usuario
tiempo_total = fin_audio_respuesta - fin_intervencion_usuario
```

En el pipeline modular se guardarán además los tiempos de STT, LLM y TTS. En un modelo S2S nativo se guardarán los eventos equivalentes que proporcione su API. Cuando una API no permita observar un evento, se anotará como no disponible y no se estimará de forma artificial.

## 9. Diseno experimental

### Variables independientes

- Plataforma o proveedor.
- Tipo de arquitectura: S2S nativa o pipeline.
- Entorno de ejecución: local con CPU, local con GPU o Google Colab con CUDA.
- Idioma de la prueba: español como principal; inglés y portugués como secundarios si son viables.
- Modelo, versión, cuantización y backend TTS.
- Tipo de tarea.
- Duracion de la conversacion.
- Presencia o ausencia de ruido.
- Tipo de conexion de red, si se estudia su efecto.

### Variables dependientes

- Latencia.
- TTFA y tiempo hasta la respuesta completa.
- WER y CER.
- Calidad de respuesta.
- Naturalidad percibida.
- Tasa de interrupciones correctas.
- Tasa de error.
- Coste estimado.
- Consumo de CPU, RAM y GPU en la linea local.

### Control experimental

- Mantener las mismas instrucciones del agente.
- Utilizar el mismo microfono y dispositivo de reproduccion.
- Fijar la calidad y el formato de audio.
- Ejecutar las pruebas en franjas comparables.
- Separar pruebas de calentamiento de las mediciones validas.
- Repetir cada escenario varias veces.
- Guardar identificadores anonimos de las ejecuciones.
- Registrar version de SDK, modelo, sistema operativo y fecha.

## 10. Tecnologias recomendadas

### Implementacion

- Python 3.12.
- Jupyter Notebook para exploracion y demostracion.
- Modulos Python separados para los adaptadores de cada proveedor.
- `asyncio` para eventos de audio y streaming.
- `python-dotenv` para configuracion local.
- `pydantic` para validar configuraciones y resultados.

### Audio y transporte

- WebRTC para comunicacion bidireccional en tiempo real.
- LiveKit para salas, transporte y gestion de participantes.
- `sounddevice` y `soundfile` para pruebas locales.
- `numpy` para inspeccion basica de senales de audio.
- Silero VAD para deteccion de turnos en la linea base local.

### Modelos y herramientas open source

- `faster-whisper` o `whisper.cpp` para STT.
- Ollama con un modelo LLM abierto para el pipeline modular.
- Piper o Kokoro para TTS.
- Moshi, Qwen2.5-Omni u otro modelo abierto para S2S nativo.
- Pipecat, LiveKit Agents o Python asincrono para la orquestacion.

No se deben mezclar implementaciones especificas de un proveedor en el nucleo de la evaluacion. El nucleo debe trabajar con una interfaz comun, por ejemplo:

```python
class VoiceAgentAdapter:
    async def connect(self) -> None: ...
    async def send_audio(self, audio_chunk: bytes) -> None: ...
    async def receive_events(self): ...
    async def close(self) -> None: ...
```

### Linea base local

- `faster-whisper` o `whisper.cpp` para STT.
- Ollama con un modelo fijado, por ejemplo Qwen.
- Piper o Kokoro para TTS.
- LiveKit autohospedado, Pipecat o audio local.

### Medicion y analisis

- `time.perf_counter_ns()` para medir latencias.
- `pandas` para consolidar resultados.
- `numpy` para estadistica basica.
- `scipy` para intervalos y contrastes cuando sean necesarios.
- `matplotlib` o `plotly` para graficos.
- `jiwer` para WER y CER.
- `psutil` para CPU y memoria.
- `GPUtil` o herramientas del fabricante para metricas de GPU.
- JSONL o Parquet para guardar resultados estructurados.

## 11. Arquitectura del software

```text
Notebook de experimentacion
        |
        v
Interfaz comun VoiceAgentAdapter
        |
        +--> Adaptador pipeline STT-LLM-TTS local
        +--> Adaptador modelo S2S abierto
        |
        v
Captura de eventos y metricas
        |
        v
Resultados JSONL/Parquet
        |
        v
Analisis estadistico y graficos
```

Cada ejecucion debe producir un registro con:

- Identificador de prueba.
- Proveedor y modelo.
- Versiones de software.
- Fecha y hora.
- Parametros de audio.
- Tiempos de cada evento.
- Resultado de la transcripcion.
- Respuesta final.
- Errores.
- Coste estimado.

## 12. Fases del TFM

### Fase 1: Revision bibliografica y del estado del arte

- Definir Speech-to-Speech, STT-LLM-TTS y agentes realtime.
- Revisar trabajos sobre latencia, turn-taking y evaluacion de voz.
- Analizar documentacion y licencias de las plataformas.
- Delimitar el vocabulario tecnico.

**Resultado:** marco teorico y criterios iniciales.

### Fase 2: Definicion del protocolo

- Seleccionar como minimo un pipeline modular y un candidato S2S abierto.
- Fijar la separación entre pruebas locales y pruebas en Colab/CUDA.
- Fijar español como idioma principal y los idiomas secundarios que sean viables.
- Fijar tareas, frases y condiciones de prueba.
- Definir eventos de latencia, métricas, número de repeticiones y método de agregación.
- Diseñar el esquema de datos de las mediciones.

**Resultado:** protocolo experimental aprobado antes de medir.

### Fase 3: Implementacion de la infraestructura

- Crear la interfaz comun de adaptadores.
- Implementar captura, reproduccion y registro de eventos.
- Separar credenciales y configuracion del codigo.
- Añadir pruebas de conectividad y errores.

**Resultado:** infraestructura reproducible.

### Fase 4: Implementacion de arquitecturas

- Implementar o consolidar el pipeline local `faster-whisper -> Ollama -> Piper/Kokoro`.
- Integrar el modelo S2S abierto que supere la fase de viabilidad.
- Mantener instrucciones y tareas equivalentes cuando la arquitectura lo permita.
- Verificar que las dos implementaciones cumplen el mismo contrato de evaluacion.

**Resultado:** agentes ejecutables bajo el mismo protocolo.

### Fase 5: Pruebas piloto

- Ejecutar pocas pruebas por arquitectura.
- Detectar problemas de audio, permisos, limites o reconexion.
- Ajustar el protocolo sin cambiar las metricas principales.
- Documentar cualquier cambio.

**Resultado:** protocolo congelado para la evaluacion final.

### Fase 6: Evaluacion final

- Ejecutar todas las tareas y repeticiones.
- Guardar datos brutos y logs.
- Registrar costes y versiones.
- Evaluar la muestra subjetiva con participantes, si procede.

**Resultado:** conjunto de datos experimental.

### Fase 7: Analisis

- Calcular estadisticos descriptivos.
- Comparar arquitecturas por metrica.
- Analizar casos de fallo.
- Separar resultados objetivos de opiniones subjetivas.
- Estudiar compromisos entre calidad, latencia, consumo, reproducibilidad y privacidad.

**Resultado:** tablas, graficos y conclusiones.

### Fase 8: Redaccion y defensa

- Explicar metodologia y limitaciones.
- Justificar la seleccion de tecnologias.
- Presentar resultados reproducibles.
- Formular la recomendacion final.
- Preparar una demostracion controlada.

**Resultado:** memoria, codigo y material de defensa.

## 13. Criterios de exito

El TFM se considerara completo si:

- Se comparan al menos un pipeline STT-LLM-TTS y un pipeline S2S open source bajo condiciones equivalentes.
- Existe una interfaz comun o un procedimiento comun de ejecucion.
- Cada resultado incluye latencia, calidad, consumo y condiciones de la prueba.
- Se reportan media, mediana, desviación estándar y percentil 95 de las latencias por configuración, idioma y entorno.
- Se distinguen los resultados de ejecución local y Google Colab/CUDA.
- Se indica el idioma principal y se justifican los idiomas secundarios incluidos o excluidos.
- Se mide al menos una metrica de reconocimiento para el pipeline modular y una metrica de calidad conversacional para ambas arquitecturas.
- Se prueban las interrupciones y los errores de conexion.
- Se incluye una discusion de privacidad, licencias y dependencia del proveedor.
- Se documentan versiones, fecha de consulta y limitaciones.
- Se proporciona una linea base local o se justifica por que no se incluye.

## 14. Riesgos y mitigaciones

| Riesgo | Mitigacion |
|---|---|
| Cambios de API o precios | Registrar fecha, version y configuracion; conservar resultados brutos |
| Coste de servicios comerciales | Mantenerlos en el marco teorico y realizar la evaluacion principal con modelos open source |
| Diferencias entre arquitecturas | Clasificar S2S nativo y pipeline por separado |
| Dependencia de la red | Medir condiciones de red o declarar la limitacion |
| Pocas muestras subjetivas | Usar rubrica clara y separar resultados exploratorios |
| Exposicion de credenciales | Usar `.env`, no imprimir secretos y mantenerlo fuera de Git |
| Datos personales en audio | Usar frases controladas y consentimiento; evitar datos sensibles |
| Dificultad para reproducir resultados | Fijar versiones, modelos, prompts, hardware y fecha |
| Soporte insuficiente para espanol | Incluir pruebas especificas de pronunciacion, nombres y acentos |

## 15. Entregables previstos

1. Memoria del TFM.
2. Documento del protocolo experimental.
3. Notebook de demostracion y analisis.
4. Adaptadores de las arquitecturas seleccionadas.
5. Linea base local reproducible.
6. Datos anonimizados de las mediciones.
7. Graficos y tablas finales.
8. Registro de versiones, costes y licencias.
9. Demostracion del agente seleccionado.

## 16. Decision tecnica inicial

La propuesta inicial es:

```text
Pipeline STT-LLM-TTS:
faster-whisper -> Ollama -> Piper/Kokoro

Agente S2S nativo:
Moshi, Qwen2.5-Omni u otro modelo abierto compatible

Transporte y orquestacion:
WebRTC, LiveKit o Pipecat, segun el modelo seleccionado

Analisis:
pandas + scipy + jiwer + matplotlib/plotly
```

OpenAI Realtime API, Gemini Live API y ElevenLabs Conversational AI quedaran documentados como referencias comerciales del estado del arte, pero no seran necesarios para ejecutar la comparativa principal. La seleccion definitiva del modelo S2S debe confirmarse despues de comprobar disponibilidad, soporte para espanol, requisitos de hardware y licencias.
