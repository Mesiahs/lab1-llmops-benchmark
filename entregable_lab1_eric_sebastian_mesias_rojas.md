# Entregable Laboratorio 1: Benchmarking de Serving Local y Cuantización

- **Curso:** LLMOps, Serving Local y Arquitecturas RAG
- **Estudiante:** Eric Sebastian Mesias Rojas
- **Hardware Utilizado:** intel core i5-7300 HQ, RAM 16GB 2400 MHZ. WINDOWS 10
- **Fecha:** 16 de noviembre de 2026

---

## Análisis Técnico de Resultados

### 1. Impacto en Memoria (MEM USAGE y KV Cache)
> ¿Cuál fue la diferencia exacta en MB/GB de RAM consumida entre `q4_K_M` y `q8_0` al ejecutar `docker stats`? ¿Por qué el consumo en ejecución supera el tamaño del archivo en disco?

**Respuesta:**
Con docker stats, el contenedor consumía 537.6 MiB en reposo (sin modelo cargado). Con q4_K_M en ejecución marcó 1.608 GiB y con q8_0 marcó 2.206 GiB. La diferencia entre ambos fue de 0.598 GiB (≈ 612 MiB, unos 0.6 GB), es decir, q4 usó ≈ 27 % menos RAM total. Descontando el consumo base, el modelo en sí ocupó ≈ 1.1 GiB en q4 y ≈ 1.7 GiB en q8. Esta diferencia coincide con la que hay en disco según ollama list (986 MB frente a 1.6 GB), porque al pasar de 8 a 4 bits los pesos ocupan casi la mitad.

El consumo en ejecución supera el tamaño del archivo porque, además de los pesos, el motor reserva memoria para el KV Cache (que almacena las claves y valores de atención de los tokens del contexto y crece con num_ctx = 2048), buffers de cómputo y la memoria propia del proceso. ollama ps lo confirma: q8_0 pesa 1.6 GB en disco pero aparece con 1.8 GB en memoria. En este modelo pequeño el KV Cache es una parte moderada de ese extra; el resto corresponde a buffers y al runtime.

---

### 2. Ancho de Banda de Memoria y Velocidad (TPS)
> En inferencia sobre CPU, el cuello de botella suele ser el ancho de banda de la memoria RAM. ¿En qué porcentaje aumentó la velocidad (TPS) al pasar de 8 bits a 4 bits?

**Respuesta:**
Los TPS medidos fueron: prompt Corto 15.11 (q4) frente a 9.97 (q8), +51.6 %; prompt Medio 14.71 frente a 10.37, +41.9 %; prompt Largo 15.08 frente a 8.79, +71.6 %. En promedio, q4_K_M generó 14.97 tok/s y q8_0 9.71 tok/s, es decir, la velocidad aumentó ≈ 54 % al pasar de 8 a 4 bits.

Este resultado es coherente con un cuello de botella de ancho de banda de memoria: para generar cada token, la CPU debe leer todos los pesos del modelo desde la RAM. Como q4 pesa ≈ 40 % menos que q8 (986 MB frente a 1.6 GB, una relación de ≈ 1.6×), hay menos datos que mover por token y la generación es más rápida. El valor del prompt Largo es el más alto porque q8 rindió peor en esa corrida (8.79 TPS), así que parte de esa diferencia puede deberse a variabilidad entre ejecuciones.

---

### 3. Latencia Inicial (TTFT) y Fase Prefill
> ¿Por qué el `TTFT (ms)` es significativamente más alto en el Prompt Largo que en el Prompt Corto?

**Respuesta:**
El TTFT del prompt Largo fue mayor que el del Corto en ambos modelos: en q4, 645.63 ms frente a 468.08 ms (+38 %); en q8, 1045.99 ms frente a 762.27 ms (+37 %). Esto se explica por la fase de prefill: antes de emitir el primer token, el modelo debe procesar todos los tokens del prompt de entrada para construir el KV Cache, y ese tiempo crece con la longitud de la entrada. El prompt Largo tiene aproximadamente el doble de caracteres que el Corto.

Cabe matizar que el TTFT más alto de todos fue el del prompt Medio (837.70 ms en q4 y 1287.73 ms en q8), que es el de entrada más larga. Por tanto, lo que determina el TTFT es la longitud del prompt de entrada, no el tipo de tarea (generar código). Además, q8_0 tuvo TTFT mayor que q4_K_M en los tres prompts, ya que mueve más datos de memoria durante el prefill.

---

### 4. Calidad vs. Eficiencia en `resultados_benchmark.md`
> Revisando las 6 respuestas completas generadas, ¿hubo alguna degradación perceptible en la lógica del código Python o en el razonamiento generado por el modelo de 4 bits frente al de 8 bits?

**Respuesta:**
Sí hubo diferencias perceptibles, aunque ninguna lo bastante clara como para atribuirla solo a la cuantización: ambos modelos (1.5B parámetros) cometieron errores serios.

Razonamiento arquitectónico: el modelo de 4 bits fue claramente peor. Recomendó CNNs, RNNs y TensorFlow para un asistente de lenguaje, confundió "modelo" con "modelo de entrenamiento" y cayó en una repetición incoherente (mencionó dos veces seguidas una solución "de Anthropic que se basa en la IA de Anthropic"). El modelo de 8 bits fue más coherente (propuso BERT/T5 de 110M-340M), aunque tampoco acertó: ninguno recomendó un modelo de 3B-8B cuantizado servido con Ollama o llama.cpp, que es lo adecuado para 16 GB de RAM sin GPU.

Código Python: la lógica de la búsqueda binaria fue correcta en ambos, pero la ejecución del código mostró incoherencias. Los dos lanzan ValueError con lista vacía mientras sus propias pruebas esperan -1. En q4 fallan 2 de 5 asserts (lista vacía y target "a", que lanza TypeError); en q8 falla 1 de 4 doctests. Ninguno implementa la verificación de lista ordenada que anuncia en su docstring, y ambos entregaron más de las 3 pruebas pedidas.

Definición de cuantización: q8 respetó mejor el formato (3 viñetas exactas) y q4 añadió introducción y sub-viñetas. Pero ninguno explicó lo esencial: reducir la precisión numérica de los pesos para ahorrar memoria y acelerar la inferencia.

Conclusión: con solo 6 respuestas y una corrida por prueba, la muestra es pequeña. q4_K_M mostró una degradación leve a moderada en razonamiento y seguimiento de formato, y q8_0 fue algo más coherente, pero la mejora de ≈ 54 % en TPS y ≈ 0.6 GB menos de RAM compensa esa diferencia modesta de calidad en un caso de uso con recursos limitados.

---
*Nota: Adjuntar junto con este documento los archivos generados `resultados_benchmark.csv` y `resultados_benchmark.md`.*
