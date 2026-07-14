# Guía de cambios — Asistente Virtual IA

**Para:** Pilar Pérez
**Proyecto:** `~/repo/asistenteTIP` (el que corre como `miapp.service` en el puerto 9000)
**Referencia:** ver `diagnostico-asistente-tip.md` para el detalle de cada falla.

Hola Pilar. Revisamos el asistente en producción con 8 preguntas típicas de estudiantes y fallaron 7. La buena noticia: casi todas las fallas tienen causas concretas y arreglables por separado. Te dejo los cambios ordenados por impacto. Hacé **un cambio por commit** y probá con las preguntas de la sección final después de cada uno.

> Importante: trabajá siempre sobre `repo/asistenteTIP`. La carpeta `~/asistenteOKCHa` es una versión vieja — te sugiero archivarla o borrarla para no editar el proyecto equivocado por error.

---

## Fase 1 — Bugs del router (backend/rag.py)

### 1.1 Quitar tildes de las keywords del router ⭐ el más urgente

**Problema:** `normalizar_consulta()` le quita las tildes a la pregunta *antes* del routing, pero las keywords de `AGENTES` las tienen ("qué días", "cuándo hay clase"). Nunca pueden coincidir → el agente de presencialidades **jamás se ejecutó** en producción (verificable en `/health`: no aparece en la lista de agentes cargados).

**Cambio:** en `detectar_agente()`, normalizá las keywords igual que la pregunta:

```python
def detectar_agente(self, pregunta: str) -> Optional[str]:
    pregunta_norm = quitar_acentos(pregunta.lower())
    puntajes = {}

    for agente, palabras in self.AGENTES.items():
        puntaje = 0
        for palabra in palabras:
            if quitar_acentos(palabra.lower()) in pregunta_norm:
                puntaje += 1
        ...
```

(O directamente escribí todas las keywords de `AGENTES` sin tildes y en minúsculas, y dejá un comentario que lo diga.)

**Prueba:** "¿Qué días tengo clases presenciales?" debe ir al agente `presencialidades` (miralo en el log `[ROUTER]`), y `presencialidades` debe aparecer en `/health` después de la primera consulta.

### 1.2 Desempatar el router con prioridad explícita

**Problema:** "¿Cuándo es el parcial de EDA?" empata 1-1 entre `horarios` ("cuando es") y `parciales` ("parcial"). `max()` resuelve el empate por orden del diccionario y gana `horarios` → el asistente responde el horario de cursada como si fuera el parcial. Es la peor falla: **una respuesta que parece bien pero contesta otra pregunta**.

**Cambio:** dale peso a las keywords fuertes en lugar de contar todas igual. Regla mínima:

```python
KEYWORDS_FUERTES = {
    "parciales": ["parcial", "parciales"],
    "docentes": ["docente", "profesor", "profesora", "quien dicta"],
    "presencialidades": ["presencial", "presencialidad", "asistencia"],
}
# en el conteo: keyword fuerte suma 10, keyword genérica ("cuando es", "hoy", "fecha") suma 1
```

Así "parcial" (10) siempre le gana a "cuando es" (1). También sacá `"hoy"` y `"mañana"` de las keywords de `horarios`: capturan casi cualquier frase.

**Prueba:** "¿Cuándo es el parcial de EDA?" debe loguear `[ROUTER] Agente detectado: parciales`.

### 1.3 Stopwords en la búsqueda por keywords (backend/agent.py)

**Problema:** la búsqueda cuenta cualquier palabra de más de 2 letras. Con "¿Cuándo empiezan las clases?", la única coincidencia en calendario fue **"las"** (de "Batalla de Las Piedras") y el modelo recibió los feriados como contexto.

**Cambio:** en `search()`, filtrá stopwords antes de puntuar:

```python
STOPWORDS = {"las", "los", "una", "uno", "del", "que", "como", "cual",
             "cuales", "para", "por", "con", "sus", "mis", "hay", "son",
             "este", "esta", "estos", "estas", "cuando", "donde", "quien"}
palabras = [p for p in query_clean.split() if len(p) > 2 and p not in STOPWORDS]
```

---

## Fase 2 — Datos (backend/data/processed/)

Estos cambios no tocan código y probablemente son los de mayor impacto en la calidad de las respuestas.

### 2.1 Agregar qué materia dicta cada docente

`docentes/contenido.txt` tiene email y horarios de consulta pero **no dice qué materia dicta cada docente**, así que "¿quién dicta X?" es incontestable. En el proyecto viejo ya existe `asistenteOKCHa/data/horarios/materias_con_profesores.txt` — usalo de fuente. Formato sugerido (una línea por materia, con sigla Y nombre completo para que matcheen las dos búsquedas):

```
Docente: Diego Piriz
Dicta: PP (Principios de Programación), EDA (Estructuras de Datos y Algoritmos)
```

### 2.2 Unificar las siglas con el diccionario MATERIAS

`parciales/contenido.txt` usa `Cont` y `P. Avanz`, pero tu normalizador convierte la pregunta a `Contab` y `ProgAvanz` → la búsqueda por sigla exacta nunca los encuentra. Regla: **toda sigla que aparezca en un contenido.txt tiene que ser una clave exacta de `MATERIAS`**. Revisá los scripts de `backend/scripts/` que generan estos archivos para que usen ese diccionario.

### 2.3 Completar los parciales que faltan

`parciales/contenido.txt` solo tiene 1er y 3er semestre. Faltan 2º, 4º, 5º y 6º (por eso "el parcial de EDA" no existe en los datos). Si esos semestres no tienen parciales este período, decilo explícitamente en el archivo ("2do semestre: sin parciales en este período") para que el asistente responda eso en lugar de "no encontré".

### 2.4 Escribir el calendario con el vocabulario del estudiante

Nadie pregunta "¿primer semestre?"; preguntan "¿cuándo empiezan las clases?". Agregá líneas redundantes con lenguaje natural:

```
Inicio de clases del primer semestre: lunes 9 de marzo de 2026
Fin de clases del primer semestre: viernes 3 de julio de 2026
Inicio de clases del segundo semestre: lunes 3 de agosto de 2026
...
```

La redundancia acá es una ventaja: le da a la búsqueda por keywords más superficie para matchear.

---

## Fase 3 — Cambios estructurales (cuando la Fase 1 y 2 estén probadas)

### 3.1 Eliminar el RAG para todo salvo cursos ⭐ el que más simplifica

Tus datos de calendario + horarios + parciales + docentes + presencialidades suman **~18 KB (~5.000 tokens)**. Eso entra cómodo en el contexto del modelo. En lugar de rutear + buscar chunks (las dos fuentes principales de error), armá el prompt con **todo ese contenido siempre**:

```python
# al iniciar, cargar los 5 archivos chicos completos
contexto_fijo = "\n\n".join(
    Path(f"data/processed/{c}/contenido.txt").read_text(encoding="utf-8")
    for c in ["calendario", "horarios", "parciales", "docentes", "presencialidades"]
)
```

y dejá la búsqueda semántica **solo** para `cursos` (142 KB, ese sí no entra). Con esto desaparecen el router, los empates y la mayoría del código de `detectar_agente`/`search`.

### 3.2 El modelo se mantiene: gemma4:e4b

Cambiar de modelo **no es una opción** en este proyecto, y no hace falta: en las pruebas, 6 de las 7 fallas ocurrieron *antes* de que el modelo viera nada (routing, búsqueda y datos). Cuando el e4b recibió el fragmento correcto (horario de BD1), respondió bien. Para que rinda al máximo:

- El punto 3.4 (suavizar el prompt) pasa a ser **obligatorio**, no opcional: el e4b es literal y con el prompt actual se rinde ante la mínima ambigüedad.
- Al aplicar 3.1 (contexto completo en el prompt), verificá con el set de pruebas que el e4b encuentre bien la respuesta dentro de los ~18 KB.

### 3.3 Memoria por sesión, no global

Hoy hay una sola instancia de `RAGSystem` y `last_topic`/`last_entity` se comparten **entre todos los usuarios**: si dos personas usan el asistente a la vez, el "sus" de una se reemplaza por la materia de la otra. Generá un `session_id` en el frontend (por ejemplo `crypto.randomUUID()` guardado en `sessionStorage`), mandalo en el body de `/ask`, y guardá la memoria en un diccionario `{session_id: {"last_topic": ..., "last_entity": ...}}` con expiración.

### 3.4 Suavizar el prompt

Reemplazá "elegí solamente la línea que responda exactamente" por algo como:

```
- Respondé usando la información del contexto.
- Si el contexto tiene la respuesta parcial, dala e indicá qué falta.
- Solo si no hay NADA relacionado, respondé: "No encontré esa información,
  te sugiero consultar en Secretaría."
```

---

## Set de pruebas — correr después de CADA cambio

Con el servidor corriendo, probá estas preguntas (hoy fallan todas menos la primera):

```bash
# desde el servidor:
curl -sN -X POST http://localhost:9000/ask -H "Content-Type: application/json" \
     -d '{"question":"¿Qué horario tiene Bases de Datos 1?"}'
```

| Pregunta | Respuesta esperada |
|----------|-------------------|
| ¿Qué horario tiene Bases de Datos 1? | Jueves 18–21, Lunes 21–24 (ya funciona — no romperla) |
| ¿Quién dicta Bases de Datos 1? | Nombre del docente (requiere 2.1) |
| ¿Cuándo es el parcial de EDA? | Fecha del parcial o "sin parcial este período" — NUNCA el horario de cursada |
| ¿Qué días tengo clases presenciales? | Datos de presencialidades |
| ¿Cuándo empiezan las clases? | 9 de marzo (1er sem.) / 3 de agosto (2do sem.) |
| ¿Qué materias tiene el tercer semestre? | BD2, COE, Contabilidad, Redes, Programación Avanzada |
| ¿Qué previas necesito para cursar Redes? | Previas según el programa, o aclarar que no está en los datos |


Cualquier duda me escribís. — Bruno
