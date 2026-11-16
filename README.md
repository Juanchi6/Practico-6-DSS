# Práctico 6 — Análisis estático de código

## 1. Ejecución y resultados de los escaneos

Se ejecutó el mismo comando desde la raíz del repositorio en cada rama. Los archivos JSON se guardaron fuera del árbol del repositorio.

| Rama | Comando |
| --- | --- |
| `main` | `semgrep scan --config auto --json --output ../semgrep-main.json .` |
| `practico-2` | `semgrep scan --config auto --json --output ../semgrep-practico-2.json .` |

| Dato informado por Semgrep | `main` | `practico-2` |
| --- | ---: | ---: |
| Versión de la CLI | 1.185.0 | 1.185.0 |
| Reglas ejecutadas | 461 | 461 |
| Archivos analizados | 62 | 63 |
| Hallazgos | 12 | 11 |
| Líneas analizadas correctamente | ~99,8 % | ~99,9 % |
| Archivos omitidos por patrones `.semgrepignore` | 1 | 1 |

Ambos escaneos finalizaron correctamente y se limitaron a archivos seguidos por Git. En los dos JSON se registró una advertencia de análisis parcial de una plantilla HTML; se tendrá en cuenta al evaluar posibles omisiones. Los términos `ERROR`, `WARNING` y `blocking` son etiquetas de la herramienta, no una confirmación automática de que cada hallazgo sea una vulnerabilidad explotable.

**Identificadores de commit analizados:** pendientes de registrar para fijar exactamente la versión de cada rama.

## 2. Herramienta: funcionamiento y limitaciones

Se usó **Semgrep Community Edition (CE) 1.185.0**. Examina el código sin ejecutar las aplicaciones y aplica reglas para localizar patrones y flujos de datos potencialmente inseguros. Con `--config auto` se cargaron 461 reglas comunitarias adecuadas al proyecto.

La edición Community permite ejecutar reglas locales y comunitarias. Frente a Semgrep Code comercial, no incluye reglas Pro ni análisis avanzado entre archivos y funciones; el escaneo OSS tampoco activó Semgrep Supply Chain. Por eso, una vulnerabilidad puede existir aunque no aparezca una alerta.

**Fuentes:** [Semgrep CE](https://semgrep.dev/products/community-edition/) · [CE y plataforma](https://semgrep.dev/products/semgrep-vs-ce/) · [Uso de `--config auto`](https://semgrep.dev/docs/semgrep-ci/sample-ci-configs).

## 3. Comparación inicial de las ramas

Al comparar regla, archivo y línea en los JSON, **10 alertas aparecen en ambas ramas**. Hay **2 alertas exclusivas de `main`** y **1 exclusiva de `practico-2`**. Estas cifras explican la diferencia neta de 12 a 11; no implican por sí solas que solo haya cambiado una vulnerabilidad.

La identificación y clasificación de las alertas se completará tras revisar el código y el contexto de cada una.

## 4. Análisis de los hallazgos

Un **falso positivo** es una alerta que no corresponde a la vulnerabilidad indicada al revisar el contexto. Un **falso negativo** es una vulnerabilidad presente que la herramienta no detectó.

### Ejercicio 1 — Inyección SQL

| Rama | Evidencia en `Ejercicio1/app.py` | Resultado de Semgrep | Clasificación |
| --- | --- | --- | --- |
| `main` | La búsqueda inserta `query`, recibido del usuario, directamente en `WHERE peliculas.nombre LIKE '%{query}%'`. | No emitió alerta SQL para ese código. | **Falso negativo:** la entrada puede alterar la consulta. |
| `practico-2` | Usa `WHERE peliculas.nombre LIKE ?` y pasa `(f'%{query}%',)` como parámetro. La columna se elige de un diccionario fijo y la dirección solo admite `ASC` o `DESC`. | Emitió una alerta SQL en la línea 44. | **Falso positivo:** la entrada no se inserta como sintaxis SQL en este flujo. |

La alerta de la rama corregida usa una regla denominada `sqlalchemy-execute-raw-query`, aunque este archivo usa `sqlite3`. La construcción dinámica de `ORDER BY` puede parecer riesgosa al analizar una sola línea, pero sus dos componentes están limitados a valores permitidos.

Para clasificar ambos casos se siguió el recorrido de `query` hasta `db.execute()`: en `main` forma parte del texto de la consulta, mientras que en `practico-2` llega como parámetro. Por eso, el número de alertas SQL por sí solo no mide la mejora entre ramas.

### Ejercicio 2 — XSS almacenado

| Rama | Evidencia | Resultado de Semgrep | Clasificación |
| --- | --- | --- | --- |
| `main` | `Ejercicio2/app.py` guarda la descripción enviada por el formulario. En `templates/edit.html:86` se muestra con `{{ pelicula['descripcion'] | safe }}`, que desactiva el escape HTML para ese valor. | No detectó el XSS en esa línea. | **Falso negativo:** el XSS existe, pero Semgrep no lo alertó. |
| `practico-2` | La misma línea muestra `{{ pelicula['descripcion'] }}`, sin `safe`. | No alertó sobre esa línea; aquí la ausencia de alerta es esperable. | La causa de este XSS fue corregida. No hay falso negativo en esa línea. |

En **ambas ramas**, Semgrep sí avisó sobre uso directo de Jinja2 en `Ejercicio2/test_seguridad.py:15`. Esa línea pertenece al **test**, que configura `select_autoescape(["html"])`; la alerta no identifica el XSS de la aplicación. La clasificamos como **falso positivo para el riesgo que sugiere en ese archivo**. El test usa la plantilla real y comprueba el comportamiento vulnerable en `main` y el escapado en `practico-2`.

La causa comprobable del XSS es `| safe` en la plantilla de la aplicación. La advertencia de análisis parcial del JSON corresponde a `templates/index.html`, no a `edit.html`; por tanto, no hay evidencia suficiente para atribuirle esa omisión.

### Ejercicio 3 — Carga de archivos

| Rama | Evidencia en `Ejercicio3/src/main/java/com/cinebuscador/controller/PeliculaController.java` | Resultado de Semgrep | Clasificación |
| --- | --- | --- | --- |
| `main` | `uploadFile` toma `getOriginalFilename()` y copia el contenido recibido a `uploadDir` sin validar tamaño, extensión ni formato. `serveFile` lo devuelve en línea con un tipo de contenido inferido del archivo. | No emitió alerta por esta carga de archivos. | **Falso negativo:** el servidor acepta y almacena archivos arbitrarios como afiches. |
| `practico-2` | Rechaza archivos vacíos y mayores a 1 MiB, permite PNG/JPG/JPEG, verifica que el contenido coincida con el formato y limita dimensiones. Guarda una imagen reconstruida con nombre UUID. Al servirla, valida el nombre y el directorio, fija el tipo de imagen y agrega `nosniff`. | Tampoco emitió alerta por esta funcionalidad. | La causa identificada en `main` fue mitigada en el código. |

La diferencia se verifica siguiendo el archivo desde `POST /upload/{id}` hasta `GET /uploads/{filename}`. Una ausencia de alertas no demuestra que una funcionalidad sea segura: en `main` Semgrep omitió controles faltantes que sí se ven al revisar el flujo completo. Esta comparación se limita a la vulnerabilidad de carga arbitraria; no certifica todas las propiedades de seguridad de `practico-2`.

En ambas ramas aparece una alerta de `django-no-csrf-token` sobre `Ejercicio3/src/main/resources/templates/upload.html`. La plantilla usa **Thymeleaf** en una aplicación **Spring**, por lo que exigir la etiqueta de Django `{% csrf_token %}` es un **falso positivo respecto de esa regla**. La alerta no permite establecer si Spring tiene o no protección CSRF: eso requiere revisar su configuración y el comportamiento de la aplicación.

### Ejercicio 4 — Inyección de expresiones SpEL

| Rama | Recorrido del dato | Resultado de Semgrep | Clasificación |
| --- | --- | --- | --- |
| `main` | `FuncionController.search` pasa el parámetro de búsqueda `buscar` a `spelEval.evaluate(buscar)`. En `SpelEvaluator.java`, el método lo procesa con `parseExpression(expression)` y `getValue(standardContext)`. | Señaló `SpelEvaluator.java:14`. | **Verdadero positivo para el flujo de búsqueda:** la entrada del usuario llega al evaluador de expresiones. |
| `practico-2` | El controlador usa `String spelResultado = buscar` y busca el texto literalmente. `SpelEvaluator` conserva el mismo código, pero este flujo ya no lo llama. | Conservó la misma alerta en `SpelEvaluator.java:14`. | **Alerta que requiere contexto:** el método señalado sigue siendo inseguro si se lo invoca con datos no confiables, pero no demuestra que el buscador corregido sea vulnerable. |

La corrección cambió el **recorrido de la entrada**, no el archivo que Semgrep señala. Por eso se mantiene una alerta en ambas ramas aunque se haya eliminado la evaluación de expresiones desde el buscador. En este flujo, la alerta de `practico-2` es un **falso positivo si se interpreta como vulnerabilidad explotable desde la búsqueda**; el método riesgoso sigue presente y convendría eliminarlo si ya no se usa.

Los tests de integración del repositorio comprueban que `1+1` se trate como texto y que una búsqueda de `Inception` devuelva una respuesta correcta. El informe se apoya aquí en el código y en esos criterios de prueba; la alerta por sí sola no prueba que se pueda alcanzar el evaluador desde la rama corregida.

**Otros hallazgos:** pendientes de validar y documentar.

## 5. Revisión estática con LLM

**Pendiente.** Se revisará el código fuente de ambas ramas con un modelo de lenguaje (LLM), registrando archivos, líneas, razonamiento y límites de cada observación. El LLM no ejecuta la aplicación ni sustituye la validación humana. Sus hallazgos se cotejarán con Semgrep y con la evidencia de los Prácticos 2 y 4.

## 6. Comparación y conclusión

**Pendiente.** Se explicará qué detectó cada método, qué omitió, qué alertas requirieron contexto y cómo cambiaron los resultados entre las ramas. La conclusión se redactará a partir de los hallazgos verificados.

## Evidencias y reproducibilidad

Los resúmenes de terminal y los archivos `semgrep-main.json` y `semgrep-practico-2.json` respaldan las cifras. La clasificación de cada caso se basa en las alertas de esos JSON y en el recorrido concreto de los datos en el código de ambas ramas.
