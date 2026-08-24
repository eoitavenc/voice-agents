# Plan de TFM: comparativa de arquitecturas de voz abiertas

## 1. Idea principal

El TFM estudiara y comparara dos arquitecturas para crear agentes de voz conversacionales:

1. Un pipeline `Speech-to-Text-to-Speech` compuesto por STT, LLM y TTS independientes.
2. Un agente `Speech-to-Speech` basado en un modelo abierto que reciba y genere audio.

La comparativa se realizara utilizando modelos y herramientas open source o con pesos disponibles para investigacion. El objetivo sera analizar que arquitectura ofrece el mejor equilibrio entre calidad, latencia, estabilidad, privacidad, consumo de recursos y facilidad de integracion.

Ya se ha probado una primera version local del pipeline `Speech-to-Text-to-Speech` con `faster-whisper`, Ollama y Piper. Esta implementacion servira como linea base del estudio. El siguiente paso sera seleccionar e integrar una alternativa `Speech-to-Speech` abierta bajo condiciones comparables.

## 2. Arquitecturas objeto de comparacion

### 2.1 Pipeline Speech-to-Text-to-Speech

Esta es la arquitectura local que ya se ha probado:

```text
Microfono -> Silero VAD -> faster-whisper -> Ollama -> Piper/Kokoro -> altavoces
```

Cada etapa puede sustituirse de forma independiente y permite medir por separado la transcripcion, la generacion y la sintesis.

### 2.2 Speech-to-Speech nativo

El modelo recibe audio y genera audio directamente:

```text
Microfono -> transporte realtime -> modelo audio-audio -> altavoces
```

Esta sera la arquitectura que se intentara seleccionar entre modelos abiertos como Moshi, Qwen2.5-Omni u otro candidato compatible con el hardware y el espanol.

## 3. Objetivo general

Diseñar y aplicar una metodologia reproducible para comparar un pipeline local Speech-to-Text-to-Speech con un agente Speech-to-Speech basado en modelos abiertos, en un escenario comun de conversacion en espanol.

## 4. Objetivos especificos

1. Revisar las plataformas comerciales realtime como contexto del estado del arte.
2. Seleccionar modelos abiertos para las arquitecturas STT-LLM-TTS y S2S.
3. Definir una arquitectura y un conjunto de tareas iguales para ambas alternativas.
4. Implementar un cliente o adaptador comun para cada arquitectura.
5. Medir objetivamente latencia, calidad, estabilidad y consumo.
6. Evaluar subjetivamente la naturalidad y la experiencia conversacional.
7. Analizar privacidad, licencias y requisitos de despliegue.
8. Proponer una recomendacion basada en los datos obtenidos.

## 5. Preguntas de investigacion

### Pregunta principal

Que arquitectura abierta ofrece el mejor equilibrio entre calidad conversacional, latencia, privacidad, consumo de recursos y facilidad de integracion para un agente de voz en espanol?

### Preguntas secundarias

- Que diferencias existen entre las soluciones Speech-to-Speech nativas y los pipelines STT-LLM-TTS?
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

### Modelos abiertos para Speech-to-Speech

- **Moshi de Kyutai**: candidato principal para el experimento audio-audio.
- **Qwen2.5-Omni**: alternativa multimodal con capacidades de audio.
- **GLM-4-Voice u otro modelo equivalente**: candidato adicional si cumple los requisitos.

La seleccion final debe considerar soporte para espanol, streaming, interrupciones, consumo de GPU, disponibilidad de pesos y licencias del codigo, los pesos, el codec y los modelos auxiliares.

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

### 7.1 Latencia

Registrar al menos:

- **Tiempo hasta el primer audio**: desde el final de la intervencion del usuario hasta el primer fragmento de respuesta audible.
- **Tiempo hasta la respuesta completa**: desde el final de la intervencion hasta que termina el audio.
- **Duracion de la respuesta**.
- **Latencia de transcripcion parcial**, si la API la proporciona.
- **Latencia de deteccion del turno**.
- **Tiempo de recuperacion despues de una interrupcion**.

Se deben guardar media, mediana, percentil 95 y desviacion estandar. La mediana representa mejor una conversacion habitual y el percentil 95 permite observar casos lentos.

### 7.2 Calidad del reconocimiento de voz

- Word Error Rate (WER), si se dispone de una transcripcion de referencia.
- Character Error Rate (CER) para nombres y palabras dificiles.
- Porcentaje de numeros, nombres y comandos transcritos correctamente.
- Errores producidos por ruido, acento o pausas.

La transcripcion de referencia debe revisarse manualmente y conservarse junto con el identificador de cada prueba.

### 7.3 Calidad de la respuesta

- Correccion de la respuesta.
- Relevancia.
- Cumplimiento de instrucciones.
- Capacidad de pedir aclaraciones.
- Consistencia entre ejecuciones.
- Tasa de respuestas incompletas o fuera de contexto.
- Tasa de errores de API y respuestas fallidas.

Se puede utilizar una rubrica de 1 a 5 puntos por criterio. Cuando se utilice evaluacion automatica con un LLM, debe complementarse con una muestra evaluada por personas y documentarse el posible sesgo.

### 7.4 Calidad de voz y conversacion

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

### 7.5 Coste y recursos

Para los pipelines open source se registrara:

- Memoria RAM y memoria de GPU utilizadas.
- Tiempo de CPU y GPU por conversacion.
- Requisitos de hardware para ejecutar cada modelo.
- Coste estimado del hardware, electricidad y mantenimiento.
- Tiempo de instalacion y configuracion.

Para las plataformas comerciales se documentara solo de forma teorica el coste por minuto, tokens, audio o sesiones, segun corresponda. Los precios se anotaran con moneda, plan, fecha de consulta y URL de la documentacion oficial. No se utilizaran para construir los resultados experimentales principales.

### 7.6 Operacion y privacidad

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

## 9. Diseno experimental

### Variables independientes

- Plataforma o proveedor.
- Tipo de arquitectura: S2S nativa o pipeline.
- Tipo de tarea.
- Duracion de la conversacion.
- Presencia o ausencia de ruido.
- Tipo de conexion de red, si se estudia su efecto.

### Variables dependientes

- Latencia.
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
- Fijar tareas, frases y condiciones de prueba.
- Definir metricas y criterios de exito.
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
- Se reportan media, mediana y percentil 95 de las latencias.
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
