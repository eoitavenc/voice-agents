# Plan de testing de modelos

Esta carpeta contiene la matriz de pruebas del pipeline de voz del TFM. No se deben descargar ni copiar aqui los pesos de los modelos: se descargan en la cache del runtime o en la ubicacion configurada por cada herramienta.

## Pipeline comun

```text
Audio -> faster-whisper/base -> LLM -> Piper -> Audio
```

Durante la primera fase se mantienen constantes `faster-whisper/base` y Piper para aislar el efecto del LLM.

## Orden recomendado

### Prueba 1: modelo principal

```text
LLM: Qwen/Qwen2.5-3B-Instruct
Runtime: Transformers
```

Esta es la prueba principal de calidad y debe ejecutarse en Colab con GPU cuando sea posible.

### Prueba 2: comparacion de runtime

```text
Modelo: Qwen2.5 3B
Runtime A: Ollama (`qwen2.5:3b`)
Runtime B: Transformers (`Qwen/Qwen2.5-3B-Instruct`)
```

La comparacion solo sera valida si se mantienen el prompt, el numero maximo de tokens, la cuantizacion, el hardware y el conjunto de preguntas.

### Prueba 3: modelo de mayor calidad

```text
LLM: Qwen/Qwen2.5-7B-Instruct
Runtime: Transformers
Cuantizacion: 4 bits si la GPU lo permite
```

Esta prueba estudia el efecto de aumentar el tamaño del modelo. No debe mezclarse con la conclusion sobre Ollama frente a Transformers.

### Pruebas opcionales

| Prioridad | Modelo | Identificador Transformers | Motivo |
|---|---|---|---|
| 1 | Qwen2.5 3B Instruct | `Qwen/Qwen2.5-3B-Instruct` | Equilibrio entre calidad y recursos |
| 2 | Qwen2.5 7B Instruct | `Qwen/Qwen2.5-7B-Instruct` | Mayor capacidad |
| 3 | Llama 3.1 8B Instruct | `meta-llama/Llama-3.1-8B-Instruct` | Referencia habitual en investigación |
| 4 | Mistral 7B Instruct | `mistralai/Mistral-7B-Instruct-v0.3` | Comparación con otra familia conocida |
| 5 | Gemma 2 9B Instruct | `google/gemma-2-9b-it` | Modelo de mayor tamaño |

Qwen2.5 1.5B se conserva como control de bajo consumo. No se considera el modelo principal de calidad.

## Backends TTS

| Prioridad | Backend | Uso |
|---|---|---|
| 1 | Piper `es_ES-davefx-medium` | Línea base estable y disponible localmente |
| 2 | F5-TTS | Prueba de naturalidad y latencia con GPU |
| 3 | Fish Speech 1.5 | Prueba multilingüe y expresiva |
| 4 | Bark | Prueba expresiva; no línea base de diálogo estable |

Los backends TTS se prueban uno a uno después de validar el pipeline con Piper.

## Casos de prueba

Guardar los audios y transcripciones de referencia fuera del código o en una carpeta de datos versionada, respetando su licencia y privacidad.

1. Pregunta factual sencilla en español.
2. Pregunta con números y nombres propios.
3. Petición que requiera seguir dos instrucciones.
4. Pregunta ambigua que deba provocar una aclaración.
5. Frase con ruido de fondo.
6. Dos turnos consecutivos sobre el mismo tema.
7. Respuesta limitada a una o dos frases.
8. Conversación corta de cinco turnos.

Cada configuración se ejecuta al menos cinco veces después de una ejecución de calentamiento.

## Métricas

Registrar por ejecución:

- Modelo, revisión y runtime.
- Hardware, GPU, RAM y VRAM.
- Versiones de Python, PyTorch y Transformers.
- Cuantización y tipo numérico.
- Tokens de entrada y salida.
- Tiempo de carga.
- Latencia del STT.
- Latencia del LLM.
- Latencia del TTS.
- Tiempo total del pipeline.
- Factor de tiempo real del audio.
- WER y CER del STT si existe referencia.
- Calidad textual de 1 a 5.
- Naturalidad, claridad y pronunciación de 1 a 5.
- Errores o respuestas truncadas.

Calcular media, mediana, desviación estándar y percentil 95. Conservar las respuestas generadas y no basar la conclusión en una única ejecución.

## Plataformas

- `Ollama` local: agente interactivo y línea base.
- `Transformers` local: control experimental, limitado por la CPU actual.
- Colab con GPU: modelos Transformer de 3B y 7B cuantizados.
- Gradio: captura del micrófono desde el navegador cuando Python se ejecuta en Colab.

La GPU, el tipo de sesión de Colab y las versiones deben quedar registrados para que el resultado sea reproducible.

## Criterio de decisión

La configuración final debe equilibrar calidad, latencia, consumo, estabilidad, privacidad, facilidad de instalación y licencia. La hipótesis inicial es que Ollama será más práctico para la conversación local y Transformers en Colab será más útil para estudiar modelos mayores y controlar el experimento.
