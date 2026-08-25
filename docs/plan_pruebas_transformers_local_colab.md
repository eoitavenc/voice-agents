# Plan de pruebas de Transformers para un agente de voz

## 1. Objetivo

Este documento define una estrategia reproducible para estudiar un pipeline local de voz y decidir cuando conviene utilizar Ollama, Transformers o una plataforma con GPU en la nube.

La pipeline comun sera:

```text
Audio -> STT -> LLM -> TTS -> Audio
```

La implementacion inicial sera modular y por turnos:

```text
faster-whisper -> modelo Transformer u Ollama -> Piper
```

F5-TTS y Fish Speech se evaluaran despues como backends TTS alternativos. No se deben cambiar varias etapas a la vez si se quiere atribuir una diferencia a un componente concreto.

## 2. Problemas detectados en la ejecucion local

### 2.1 Hardware y PyTorch

El equipo utilizado actualmente informa de:

```text
GPU: AMD Radeon integrada
Memoria grafica informada: 2 GB
PyTorch: 2.13.0+cpu
CUDA disponible: False
```

La GPU integrada no esta siendo utilizada por PyTorch. Por tanto, el LLM se ejecuta en CPU y no puede beneficiarse de la ruta habitual de CUDA y cuantizacion `bitsandbytes` para GPU NVIDIA.

### 2.2 Latencia del LLM

En las pruebas del notebook, Qwen2.5 1.5B produjo una respuesta en aproximadamente 20,8 segundos. La carga inicial del modelo requirio aproximadamente 11 segundos en una ejecucion posterior.

Estos tiempos son demasiado altos para una conversacion de voz natural, aunque sirven para validar la arquitectura y realizar una prueba de concepto.

### 2.3 Offload a disco

Con `device_map="auto"`, una ejecucion anterior mostro que algunos parametros se enviaban a CPU y disco. El offload a disco evita un error de memoria, pero introduce accesos lentos y hace que el agente sea poco usable.

La configuracion actual evita ese comportamiento en CPU y carga el modelo directamente en RAM. Esta decision tiene una consecuencia importante: si el modelo no cabe en RAM, la carga debe fallar y no ocultar el problema enviando capas al disco.

### 2.4 Modelo pequeno frente a calidad

Qwen2.5 1.5B reduce el consumo, pero su capacidad de razonamiento, seguimiento de instrucciones y conocimiento es inferior a la de modelos de 3B, 7B u 8B. El problema observado es un compromiso directo:

```text
Menor tamaño -> menor consumo y, normalmente, menor latencia
Mayor tamaño -> mayor calidad y mayor consumo
```

Reducir `max_new_tokens` acorta la respuesta, pero no convierte un modelo pequeño en uno mas capaz.

### 2.5 Jupyter y los controles de voz

Jupyter es util para experimentar, pero los widgets, callbacks y hilos pueden quedar asociados a ejecuciones anteriores. Esto produjo problemas con el boton del microfono y con la segunda iteracion.

Ademas, `sounddevice` funciona mejor cuando Python se ejecuta en el mismo equipo que el microfono. Si el notebook se ejecuta en Colab, el kernel esta en un servidor remoto y no puede acceder directamente al microfono local mediante `sounddevice`.

### 2.6 Salida de audio

`IPython.display.Audio` crea un reproductor dentro del notebook, pero no garantiza que el sonido se reproduzca por los altavoces del sistema. Para una aplicacion web, la salida debe devolverse como un componente de audio del navegador. En ejecucion local se puede usar `sounddevice` para reproducir el WAV.

### 2.7 TTS y compatibilidad de APIs

Piper ya esta integrado en el proyecto y es una buena linea base. F5-TTS y Fish Speech tienen interfaces, modelos, requisitos de hardware y licencias propios. No se debe asumir que existe una llamada comun como `tts.generate(text)` para todos ellos.

## 3. Plataforma recomendada

### 3.1 Desarrollo y linea base local

Se mantendra la ejecucion local para:

- Comparar Ollama con el notebook de Transformers.
- Probar privacidad y ausencia de red.
- Validar STT y Piper con el microfono y los altavoces reales.
- Reproducir la linea base en el mismo hardware del usuario.

La linea base local recomendada es:

```text
faster-whisper/base -> Ollama + qwen2.5:3b -> Piper
```

Para el agente interactivo, Ollama es preferible en este equipo porque utiliza un runtime local optimizado para modelos cuantizados y evita cargar directamente los pesos mediante PyTorch.

### 3.2 Experimento con GPU en Google Colab

Colab es la primera opcion para evaluar Transformers con modelos mayores. La GPU disponible puede variar entre T4, L4 u otra configuracion, por lo que cada ejecucion debe registrar el tipo de GPU.

En Colab se recomienda:

```text
faster-whisper -> Transformers cuantizado -> Piper o TTS experimental
```

La captura de audio no debe utilizar `sounddevice`. Se debe usar un archivo WAV subido o una interfaz Gradio con `gr.Audio`, que captura el microfono del navegador y envia el archivo al kernel remoto.

Colab es adecuado para:

- Ejecutar Qwen 3B o 7B en GPU.
- Probar cuantizacion de 4 bits.
- Medir memoria VRAM y latencia.
- Repetir experimentos con una configuracion documentada.

Sus limitaciones son las desconexiones, la disponibilidad variable de GPU, los limites de almacenamiento y la posible perdida de reproducibilidad entre sesiones.

### 3.3 Otras plataformas

| Plataforma | Uso recomendado | Limitacion principal |
|---|---|---|
| Google Colab | Primera plataforma con GPU para el TFM | Sesiones y GPU variables |
| Colab Pro | Experimentos largos y GPU mas estable | Coste mensual |
| Kaggle Notebooks | Repeticiones con GPU sin configurar una maquina propia | Limites de tiempo y almacenamiento |
| Hugging Face Spaces + Gradio | Demo interactiva con microfono en navegador | GPU y recursos dependen del Space |
| Maquina virtual cloud | Evaluacion controlada y servidor persistente | Coste y configuracion |
| Ejecucion local con Ollama | Uso diario, privacidad y linea base | Limitada por el hardware local |

La recomendacion general es:

```text
Ollama local = agente interactivo y linea base
Colab GPU = experimento academico de Transformers
Gradio/Space = demostracion web
```

## 4. Arquitecturas experimentales

### Configuracion A: linea base local

```text
Micrófono local -> faster-whisper/base -> Ollama qwen2.5:3b -> Piper -> altavoces
```

Sirve para establecer una referencia realista de latencia, privacidad y facilidad de uso.

### Configuracion B: runtime comparable

```text
WAV de prueba -> faster-whisper/base -> Qwen2.5-3B-Instruct con Transformers -> Piper -> WAV
```

Esta es la comparacion principal entre runtimes. Se debe usar el mismo modelo, prompt, tokenizer logico, numero maximo de tokens y hardware siempre que sea posible.

### Configuracion C: modelo de mayor calidad

```text
WAV de prueba -> faster-whisper/base -> Qwen2.5-7B-Instruct cuantizado -> Piper -> WAV
```

Se ejecutara en Colab GPU. Su objetivo es medir cuanto mejora la calidad al aumentar el tamaño del LLM y cuanto empeoran memoria y latencia.

### Configuracion D: TTS experimental

```text
WAV de prueba -> faster-whisper/base -> Transformer -> F5-TTS o Fish Speech -> WAV
```

Esta configuracion se evaluara despues de validar Piper. El TTS debe cambiarse individualmente para medir su efecto.

## 5. Modelos recomendados

### 5.1 STT

| Modelo | Identificador o API | Uso |
|---|---|---|
| faster-whisper base | `WhisperModel("base")` | Linea base equilibrada para español y CPU |
| faster-whisper small | `WhisperModel("small")` | Mayor calidad si la latencia es aceptable |
| Whisper medium | Variante de mayor coste | Solo si la GPU y el conjunto de pruebas lo justifican |

Se debe mantener el mismo STT al comparar Ollama y Transformers. Si se cambia STT, el resultado ya no mide solo el efecto del LLM.

### 5.2 LLM para Transformers

| Prioridad | Modelo | Motivo |
|---|---|---|
| 1 | `Qwen/Qwen2.5-3B-Instruct` | Mejor punto de partida para calidad, tamaño y español |
| 2 | `Qwen/Qwen2.5-7B-Instruct` | Evaluacion de mayor capacidad en Colab GPU |
| 3 | `meta-llama/Llama-3.1-8B-Instruct` | Referencia muy citada en investigacion |
| 4 | `mistralai/Mistral-7B-Instruct-v0.3` | Referencia conocida para comparar otra familia |
| 5 | `google/gemma-2-9b-it` | Modelo de mayor tamaño para una prueba adicional |

La seleccion recomendada para el experimento principal es:

```text
Qwen2.5 3B como modelo base
Qwen2.5 7B como variante de calidad
```

Qwen2.5 1.5B se conservara como prueba de bajo consumo, no como modelo principal de calidad. Llama, Mistral y Gemma deben incluirse solo si el tiempo y la GPU permiten comparaciones repetidas.

Los nombres de modelos, versiones, revisiones y licencias deben registrarse. Una afirmacion como "mejor en español" debe tratarse como hipotesis y comprobarse mediante las pruebas del TFM.

### 5.3 TTS

| Prioridad | Modelo | Uso |
|---|---|---|
| 1 | Piper `es_ES-davefx-medium` | Linea base estable, local y ya disponible |
| 2 | F5-TTS | Prueba de naturalidad y clonacion con GPU |
| 3 | Fish Speech 1.5 | Prueba multilingue y expresiva si se fijan dependencias |
| 4 | Bark | Experimento expresivo, no linea base de dialogo estable |

Piper debe utilizarse en las primeras comparaciones porque permite aislar el efecto del LLM. F5-TTS y Fish Speech se evaluaran como backends alternativos con sus propias instrucciones y formatos.

## 6. Plan de pruebas de Transformers

### Fase 0: verificacion del entorno

Registrar:

- Sistema operativo.
- Version de Python.
- Version de PyTorch y Transformers.
- Tipo de CPU y GPU.
- RAM y VRAM disponibles.
- Tipo de ejecucion: local, Colab, Kaggle o cloud.
- Modelo y revision exacta.
- Cuantizacion y tipo numerico.

### Fase 1: prueba de texto

Antes de utilizar audio, ejecutar diez preguntas textuales fijas. Esta fase permite comprobar el LLM sin introducir errores de microfono, STT o TTS.

Medir:

- Tiempo de carga.
- Latencia de generacion.
- Tokens de entrada y salida.
- Memoria utilizada.
- Respuesta vacia, truncada o con texto no deseado.

### Fase 2: prueba STT

Utilizar audios en español con transcripciones de referencia revisadas manualmente.

Medir:

- WER.
- CER para nombres y numeros.
- Tiempo de transcripcion.
- Errores con ruido y pausas.

El mismo conjunto de audios se utilizara para todas las configuraciones.

### Fase 3: pipeline con Piper

Ejecutar:

```text
Audio -> faster-whisper -> LLM -> Piper -> Audio
```

Probar primero Qwen2.5 3B con Ollama y con Transformers. Despues repetir con Qwen2.5 7B en Colab si hay GPU.

Medir por turno:

- Latencia STT.
- Latencia LLM.
- Latencia TTS.
- Tiempo total.
- Duracion del audio de salida.
- Factor de tiempo real: tiempo total dividido por duracion del audio generado.
- Tasa de ejecuciones fallidas.

### Fase 4: comparacion de modelos

Con el mismo runtime, comparar Qwen2.5 3B, Qwen2.5 7B y, si hay recursos, Llama 3.1 8B o Mistral 7B.

La variable independiente sera el modelo. El prompt, STT, TTS, hardware y conjunto de pruebas deben mantenerse constantes.

### Fase 5: comparacion de runtimes

Con el mismo modelo, comparar:

```text
Ollama local
Transformers local
Transformers en Colab GPU
```

Esta fase debe separar claramente el efecto del runtime del efecto de la GPU. No se deben comparar resultados de CPU y GPU como si fueran solo diferencias entre Ollama y Transformers.

### Fase 6: comparacion de TTS

Mantener fijo el STT y el LLM y cambiar unicamente:

```text
Piper -> F5-TTS -> Fish Speech
```

Evaluar naturalidad, pronunciacion española, pausas, estabilidad, latencia y memoria. Registrar licencias del codigo, pesos, voces y modelos auxiliares.

## 7. Protocolo de repeticion

Cada prueba debe ejecutarse al menos cinco veces despues de una ejecucion de calentamiento. Para cada configuracion guardar:

- Media.
- Mediana.
- Desviacion estandar.
- Percentil 95.
- Numero de errores.

Para comparar calidad textual se utilizara el mismo conjunto de preguntas. Para calidad de voz, al menos dos evaluadores pueden puntuar claridad, naturalidad, pronunciacion y adecuacion de la entonacion en una escala de 1 a 5.

Las respuestas generadas deben conservarse junto con el identificador de la prueba. Si se usa muestreo, se debe registrar la semilla o indicar que el resultado es estocastico. Para la comparacion inicial se recomienda `temperature=0` y `do_sample=False`.

## 8. Criterio de decision

La decision no se basara en una unica respuesta. Se elegira la configuracion considerando simultaneamente:

- Calidad de la respuesta.
- WER del STT.
- Latencia hasta el audio.
- Factor de tiempo real.
- Memoria y VRAM.
- Estabilidad.
- Privacidad.
- Facilidad de instalacion.
- Licencia y posibilidad de distribuir el sistema.

La hipotesis practica para este proyecto es:

```text
Ollama sera mas adecuado para la interaccion local.
Transformers en Colab permitira estudiar modelos mayores y controlar mejor el experimento.
```

Esta hipotesis debe confirmarse con los datos obtenidos, no presentarse como conclusion previa.

## 9. Recomendacion final de trabajo

1. Mantener `Ollama_Local_Agent.ipynb` como linea base local.
2. Mantener `Transformers_Local_Voice_Agent.ipynb` como notebook de investigacion.
3. Crear una variante Colab que utilice GPU, cuantizacion y Gradio para el microfono.
4. Usar Qwen2.5 3B como comparacion principal.
5. Usar Qwen2.5 7B como prueba de mayor calidad en Colab.
6. Mantener faster-whisper/base y Piper constantes en la primera fase.
7. Evaluar F5-TTS y Fish Speech solo despues de validar el pipeline con Piper.
8. Ejecutar al menos cinco repeticiones por configuracion y conservar los resultados.
