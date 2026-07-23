# Prompt: generar notebook de RAG para análisis de contratos

Copiá y pegá el siguiente prompt en cualquier asistente de IA (Claude, ChatGPT, etc.) para que te genere el notebook completo de una sola vez.

---

Necesito que generes un **notebook completo de Google Colab (.ipynb)**, listo para subir y ejecutar de punta a punta, que implemente un sistema **RAG (Retrieval-Augmented Generation)** para analizar contratos legales en Word.

## Entorno y restricciones

- Se ejecuta en Google Colab (runtime estándar, con o sin GPU T4).
- Debe funcionar 100% con librerías gratuitas/locales, sin API keys obligatorias (embeddings y LLM corriendo local vía HuggingFace).
- El entregable final tiene que ser un archivo `.ipynb` real (JSON válido de notebook, con `execution_count: null` como literal JSON, NO como código Python) — no un script plano ni el JSON pegado dentro de una celda de código.

## Input

- El usuario sube un contrato desde su computadora usando `google.colab.files.upload()`.
- El archivo puede venir en `.doc` (binario viejo) o `.docx` (OOXML). El notebook debe:
  - Detectar la extensión automáticamente.
  - Si es `.doc`, convertirlo a `.docx` usando LibreOffice en modo headless (`libreoffice --headless --convert-to docx`).
  - Instalar `libreoffice-writer` (no el paquete completo `libreoffice`, para reducir tiempo y riesgo de fallos) con manejo robusto de errores de apt:
    - Correr `apt-get update` con `-o Acquire::Retries=3` antes de instalar.
    - Si falla por 404 en `security.ubuntu.com`, hacer fallback reemplazando ese mirror por `archive.ubuntu.com` en `/etc/apt/sources.list` y reintentar.
    - Envolver la conversión en try/except con mensajes de error claros (path no encontrado, conversión fallida, etc.).

## Pipeline RAG

1. **Extracción de texto**: con `python-docx`, extrayendo tanto párrafos como texto dentro de tablas (los contratos suelen tener cláusulas o anexos en tablas).
2. **Chunking**: función configurable (`chunk_size`, `overlap`) que trocea el texto en fragmentos con solapamiento, para no cortar cláusulas a la mitad.
3. **Embeddings**: `sentence-transformers`, modelo `paraphrase-multilingual-MiniLM-L12-v2` (soporta español), normalizando vectores.
4. **Índice vectorial**: `faiss-cpu`, `IndexFlatIP` (similitud coseno con vectores normalizados).
5. **Retrieval**: función que recibe una pregunta, la embebe y devuelve los `top_k` chunks más relevantes con su score.
6. **Generación (LLM local)**: modelo HuggingFace `Qwen/Qwen2.5-1.5B-Instruct` (multilingüe, liviano; mencionar `Qwen2.5-0.5B-Instruct` como alternativa si falta memoria), usando `apply_chat_template` con un system prompt que:
   - Instruya al modelo a responder ÚNICAMENTE en base al contexto recuperado.
   - Le pida decir explícitamente "No encuentro esa información en el contrato" si no está en el contexto.
   - Le pida citar la cláusula o fragmento relevante cuando sea posible.
7. **Función end-to-end** `responder_sobre_contrato(pregunta, top_k, mostrar_contexto)` que encadena retrieval + prompt + generación.

## Ejemplos de uso

- Al menos 3 preguntas de ejemplo ya ejecutadas en celdas separadas (ej: partes del contrato, duración/renovación, penalidades por incumplimiento).
- Una celda opcional con un **checklist automático** de ~9 preguntas típicas de revisión de contratos (partes, objeto, duración, obligaciones, pago, terminación anticipada, penalidades, ley aplicable/jurisdicción, confidencialidad), que itera y arma un resumen en una lista de tuplas `(pregunta, respuesta)`.

## Estructura del notebook (celdas separadas, con markdown explicativo entre cada bloque de código)

1. Instalar dependencias (`python-docx`, `sentence-transformers`, `faiss-cpu`, `transformers`, `accelerate`) + LibreOffice con el manejo de errores descrito arriba.
2. Subir archivo.
3. Convertir a `.docx` si hace falta.
4. Extraer texto.
5. Chunking.
6. Embeddings + índice FAISS.
7. Función de retrieval.
8. Cargar modelo de generación.
9. Función RAG completa.
10. Ejemplos de preguntas.
11. Checklist automático (opcional).
12. Notas finales: cómo migrar a embeddings de OpenAI o a un LLM vía API (Claude/OpenAI) reemplazando solo la parte de generación, manteniendo igual el retrieval.

## Formato de salida

Devolveme directamente el JSON completo del `.ipynb` (nbformat 4), no un resumen ni fragmentos sueltos.
