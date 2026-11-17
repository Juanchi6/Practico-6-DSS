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

**Referencias de las ramas consultadas después de los escaneos:** [`main` — `9299e0bc9380fc5295da1f605670fcc0703301b5`](https://github.com/Juanchi6/Practico-2-DSS/commit/9299e0bc9380fc5295da1f605670fcc0703301b5) y [`practico-2` — `132e676cf4be0dc7c8102975fe3faf5d9949fccf`](https://github.com/Juanchi6/Practico-2-DSS/commit/132e676cf4be0dc7c8102975fe3faf5d9949fccf). Los JSON de Semgrep no registran el SHA de Git: estas referencias permiten localizar las ramas revisadas, pero los JSON por sí solos no prueban el commit exacto de cada ejecución.

## 2. Herramienta: funcionamiento y limitaciones

Se usó **Semgrep Community Edition (CE) 1.185.0**. Examina el código sin ejecutar las aplicaciones y aplica reglas para localizar patrones y flujos de datos potencialmente inseguros. Se usó `--config auto`, la opción que Semgrep CE recomienda para elegir automáticamente reglas del registro según el proyecto. En estas ejecuciones seleccionó 461 reglas por rama; no se añadieron reglas propias. Como la selección proviene del registro, un escaneo futuro podría elegir otras reglas.

La edición Community permite ejecutar reglas locales y comunitarias. Frente a Semgrep Code comercial, no incluye reglas Pro ni análisis avanzado entre archivos y funciones; el escaneo OSS tampoco activó Semgrep Supply Chain. Por eso, una vulnerabilidad puede existir aunque no aparezca una alerta.

**Fuentes:** [Semgrep CE](https://semgrep.dev/products/community-edition/) · [CE y plataforma](https://semgrep.dev/products/semgrep-vs-ce/) · [Uso de `--config auto`](https://semgrep.dev/docs/semgrep-ci/sample-ci-configs).

## 3. Comparación inicial de las ramas

Al comparar regla, archivo y línea en los JSON, **10 alertas aparecen en ambas ramas**. Hay **2 alertas exclusivas de `main`** y **1 exclusiva de `practico-2`**. Estas cifras explican la diferencia neta de 12 a 11; no implican por sí solas que solo haya cambiado una vulnerabilidad.

Las diez alertas comunes se distribuyen así: cinco por Dockerfile sin `USER`, tres de la regla CSRF de Django aplicada a Thymeleaf, una sobre Jinja2 en un test y una sobre el evaluador SpEL que permanece en el código. Las dos alertas exclusivas de `main` corresponden a AES/ECB; la exclusiva de `practico-2`, a SQL. El análisis del código también identificó vulnerabilidades que no figuran entre las alertas.

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
| `main` | `Ejercicio2/app.py` guarda la descripción enviada por el formulario. En `templates/edit.html:86` se muestra con el filtro `safe`, que desactiva el escape HTML para ese valor. | No detectó el XSS en esa línea. | **Falso negativo:** el XSS existe, pero Semgrep no lo alertó. |
| `practico-2` | La misma línea muestra `{{ pelicula['descripcion'] }}`, sin `safe`. | No alertó sobre esa línea; aquí la ausencia de alerta es esperable. | La causa de este XSS fue corregida. No hay falso negativo en esa línea. |

En **ambas ramas**, Semgrep sí avisó sobre uso directo de Jinja2 en `Ejercicio2/test_seguridad.py:15`. Esa línea pertenece al **test**, que configura `select_autoescape(["html"])`; la alerta no identifica el XSS de la aplicación. La clasificamos como **falso positivo para el riesgo que sugiere en ese archivo**. El test carga la plantilla real y está diseñado para fallar ante el filtro `safe` de `main` y pasar con el escape de `practico-2`; aquí se describe el criterio escrito en el test, sin atribuirle una ejecución nueva.

La causa comprobable del XSS es `| safe` en la plantilla de la aplicación. La advertencia de análisis parcial del JSON corresponde a `templates/index.html`, no a `edit.html`; por tanto, no hay evidencia suficiente para atribuirle esa omisión.

### Ejercicio 3 — Carga de archivos

| Rama | Evidencia en `Ejercicio3/src/main/java/com/cinebuscador/controller/PeliculaController.java` | Resultado de Semgrep | Clasificación |
| --- | --- | --- | --- |
| `main` | `uploadFile` toma `getOriginalFilename()` y copia el contenido recibido a `uploadDir` sin validar tamaño, extensión ni formato. `serveFile` lo devuelve en línea con un tipo de contenido inferido del archivo. | No emitió alerta por esta carga de archivos. | **Falso negativo:** el servidor acepta y almacena archivos arbitrarios como afiches. |
| `practico-2` | Rechaza archivos vacíos y mayores a 1 MiB, permite PNG/JPG/JPEG, verifica que el contenido coincida con el formato y limita dimensiones. Guarda una imagen reconstruida con nombre UUID. Al servirla, valida el nombre y el directorio, fija el tipo de imagen y agrega `nosniff`. | Tampoco emitió alerta por esta funcionalidad. | La causa identificada en `main` fue mitigada en el código. |

La diferencia se verifica siguiendo el archivo desde `POST /upload/{id}` hasta `GET /uploads/{filename}`. Una ausencia de alertas no demuestra que una funcionalidad sea segura: en `main` Semgrep omitió controles faltantes que sí se ven al revisar el flujo completo. Esta comparación se limita a la vulnerabilidad de carga arbitraria; no certifica todas las propiedades de seguridad de `practico-2`.

La alerta de Semgrep sobre el formulario de carga de archivos se analiza junto con las otras alertas de la misma regla en la sección «Alertas de CSRF — Regla de Django aplicada a Spring».

### Ejercicio 4 — Inyección de expresiones SpEL

| Rama | Recorrido del dato | Resultado de Semgrep | Clasificación |
| --- | --- | --- | --- |
| `main` | `FuncionController.search` pasa el parámetro de búsqueda `buscar` a `spelEval.evaluate(buscar)`. En `SpelEvaluator.java`, el método lo procesa con `parseExpression(expression)` y `getValue(standardContext)`. | Señaló `SpelEvaluator.java:14`. | **Verdadero positivo para el flujo de búsqueda:** la entrada del usuario llega al evaluador de expresiones. |
| `practico-2` | El controlador usa `String spelResultado = buscar` y busca el texto literalmente. `SpelEvaluator` conserva el mismo código, pero este flujo ya no lo llama. | Conservó la misma alerta en `SpelEvaluator.java:14`. | **Alerta que requiere contexto:** el método señalado sigue siendo inseguro si se lo invoca con datos no confiables, pero no demuestra que el buscador corregido sea vulnerable. |

Aunque el evaluador construye un `SimpleEvaluationContext`, ejecuta la expresión con `StandardEvaluationContext`, al que además añade variables relacionadas con `System` y `Runtime`. La corrección cambió el **recorrido de la entrada**, no el archivo que Semgrep señala. Por eso se mantiene una alerta en ambas ramas aunque se haya eliminado la evaluación de expresiones desde el buscador. En este flujo, la alerta de `practico-2` es un **falso positivo si se interpreta como vulnerabilidad explotable desde la búsqueda**; el método riesgoso sigue presente y convendría eliminarlo si ya no se usa.

Los tests de integración del repositorio están diseñados para exigir que `1+1` se trate como texto y que una búsqueda de `Inception` responda correctamente. Esta sección se apoya en el código y en esos criterios, sin atribuirles una ejecución nueva; la alerta por sí sola no prueba que se pueda alcanzar el evaluador desde la rama corregida.

### Ejercicio 5 — Protección de contraseñas

| Rama | Evidencia en el código | Resultado de Semgrep | Clasificación |
| --- | --- | --- | --- |
| `main` | `AuthController` guarda la contraseña con `EncryptionService.encrypt()` y la recupera con `decrypt()` para comparar durante el inicio de sesión. El servicio usa `AES/ECB/PKCS5Padding` y define una clave fija en el código fuente. | Emitió **dos alertas** sobre AES/ECB en `EncryptionService.java:50` y `:64`, correspondientes a cifrar y descifrar. No alertó sobre la clave fija ni sobre el almacenamiento reversible de contraseñas. | **Verdaderos positivos** para el modo ECB; **falsos negativos** para la clave embebida y el diseño de contraseñas reversibles. |
| `practico-2` | `EncryptionService` genera hashes `Argon2id` con sal y verifica contraseñas mediante `matches()`. El controlador ya no descifra las contraseñas. | Desaparecieron las dos alertas AES/ECB. | El cambio elimina el cifrado reversible y la clave fija de este servicio; no hay alerta ECB porque ya no se usa AES/ECB allí. |

Las dos alertas de `main` corresponden a **dos usos del mismo modo inseguro**, no a dos causas independientes. Además, una contraseña almacenada con cifrado reversible puede recuperarse si se obtiene la clave; con hashes para contraseñas, la verificación compara la contraseña ingresada con el hash sin descifrarla. Semgrep detectó el patrón `AES/ECB`, pero el análisis del servicio y de `AuthController` permitió identificar los problemas adicionales que omitió.

Los tests incluidos en `EncryptionServiceTest.java` definen dos condiciones para la corrección: que el valor guardado comience con `$argon2id$` y que dos usos de la misma contraseña produzcan hashes distintos. Esta sección describe lo que comprueban esos tests, sin atribuirles resultados de ejecución nuevos.

### Configuración de Docker — Cinco alertas comunes

Semgrep señaló **un Dockerfile por ejercicio** en `main` y en `practico-2` porque ninguno declara `USER`. Los cinco Dockerfile son iguales entre ramas; los archivos `docker-compose.yml` correspondientes tampoco establecen un usuario mediante `user:`.

| Archivos afectados | Qué indica la alerta | Clasificación |
| --- | --- | --- |
| `Ejercicio1/Dockerfile` y `Ejercicio2/Dockerfile` | La imagen final ejecuta Flask sin declarar un usuario en el Dockerfile ni en Compose. | **Alerta válida de configuración**, presente en las dos ramas. |
| `Ejercicio3/Dockerfile`, `Ejercicio4/Dockerfile` y `Ejercicio5/Dockerfile` | La imagen final ejecuta Java sin declarar un usuario en el Dockerfile ni en Compose. | **Alerta válida de configuración**, presente en las dos ramas. |

La alerta advierte que no se asigna explícitamente una identidad con menos privilegios al proceso de la aplicación. Los archivos analizados justifican la advertencia, pero **no incluyen una comprobación del usuario efectivo dentro del contenedor**, que haría falta para afirmar con certeza qué identidad tiene en ejecución.

**Cambios necesarios para corregirla:** definir un usuario del sistema sin privilegios en la imagen final, usar `USER` antes de `CMD` o `ENTRYPOINT` y preparar los permisos de escritura. En los Ejercicios 1 y 2, `init_db.py` crea `cine.db` y `cine2.db` dentro de `/app`. En los Ejercicios 3 a 5, la configuración de Docker dirige la base H2 hacia `/data/cine3`; además, el Ejercicio 3 escribe afiches en `/uploads`. El Dockerfile del Ejercicio 3 crea `/data` y `/uploads`, mientras que los de los Ejercicios 4 y 5 no crean `/data`, por lo que allí también habría que verificar que el directorio exista.

Este cambio **no agrega autenticación a la aplicación**: el usuario de `USER` identifica al proceso dentro del contenedor y no corresponde a una persona que inicia sesión. Tras ajustar los permisos, se reconstruirían y levantarían los contenedores con Docker Compose como en las pruebas originales; habría que comprobar que cada ejercicio inicia, puede crear o abrir su base de datos y, en el Ejercicio 3, puede guardar afiches. Un `USER` agregado sin esos ajustes puede producir errores de permisos. Este informe analiza las alertas; no se modificaron los Dockerfile ni se ejecutaron estas verificaciones.

Estas cinco alertas son **la mitad de las diez alertas comunes** a ambas ramas; permanecen porque las correcciones de los ejercicios no cambiaron la configuración de Docker. La [referencia oficial de Docker para `USER`](https://docs.docker.com/reference/dockerfile/#user) especifica que determina el usuario de ejecución de `CMD` y `ENTRYPOINT`.

### Alertas de CSRF — Regla de Django aplicada a Spring

CSRF es un ataque que intenta hacer que el navegador de una persona envíe una petición no deseada a una aplicación en la que tiene una sesión activa. Semgrep encontró tres formularios sin la etiqueta `{% csrf_token %}`: uno de carga de archivos en `Ejercicio3/src/main/resources/templates/upload.html:39` y dos de login y registro en `Ejercicio5/src/main/resources/templates/index.html:45` y `:56`.

**Clasificación:** las tres alertas aparecen en ambas ramas y son **falsos positivos de la regla `django-no-csrf-token`**. Esa regla exige una etiqueta propia de **Django**, pero los tres formularios están hechos con **Thymeleaf para Spring**. Por eso, la ausencia de esa etiqueta no demuestra una vulnerabilidad en estas aplicaciones.

**Alcance:** esta clasificación se refiere solo a la regla de Django. No se comprobó por separado si los formularios de Spring están protegidos frente a CSRF; sería necesario revisar la protección de la aplicación y probar su comportamiento para concluirlo.

## 5. Revisión del código con LLM y comparación por ejercicio

Un **LLM** es un modelo de lenguaje capaz de examinar y explicar código. Este asistente revisó archivos de `main` y `practico-2` **después de conocer los JSON de Semgrep**. Por eso, la revisión no fue ciega ni constituye una prueba independiente de cuál método detecta más. Se analizaron rutas de entrada, consultas, plantillas, controladores, contraseñas y configuración de los cinco ejercicios. En esta etapa no se ejecutaron pruebas nuevas.

### Ejercicio 1 — Inyección SQL

| Rama | Semgrep | Revisión del LLM |
| --- | --- | --- |
| `main` | No alertó sobre la consulta de `Ejercicio1/app.py`. | `buscar` llega a `query` y se inserta en `LIKE '%{query}%'`: **inyección SQL omitida por Semgrep**. |
| `practico-2` | Alertó sobre `app.py:44`. | `query` se pasa a `db.execute()` como parámetro y el orden se limita a valores admitidos: **falso positivo para esta consulta**. |

En ambas ramas Semgrep también señaló que `Ejercicio1/Dockerfile` no declara `USER`. El LLM corroboró la configuración; la identidad efectiva del proceso no se comprobó ejecutando el contenedor.

### Ejercicio 2 — XSS e inyección SQL que permanece

| Rama | Semgrep | Revisión del LLM |
| --- | --- | --- |
| `main` | Alertó sobre Jinja2 en `test_seguridad.py`, pero no sobre la plantilla ni la búsqueda SQL. | `templates/edit.html:86` muestra la descripción con el filtro `safe`: **XSS omitido**. En `app.py:31-35`, `query` y `sort_dir` entran directamente en el SQL: **inyección SQL también omitida**. |
| `practico-2` | Mantuvo la alerta en el test y no señaló la consulta SQL. | La plantilla ya no usa el filtro `safe`, por lo que corrigió ese XSS. `app.py` es idéntico al de `main`: **la inyección SQL continúa en esta rama**. |

La alerta del test no identifica el XSS de la aplicación: el propio test configura escape automático. En ambas ramas Semgrep también alertó sobre `Dockerfile` sin `USER`. El LLM observó además que `app.py` inicia Flask con `debug` activado por defecto y escucha en `0.0.0.0`; el mismo comportamiento aparece en el Ejercicio 1. Es una configuración riesgosa si se expone fuera del entorno de práctica, sin prueba de acceso externo. [Flask desaconseja usar su depurador en producción](https://flask.palletsprojects.com/en/stable/debugging/).

### Ejercicio 3 — Carga de archivos

| Rama | Semgrep | Revisión del LLM |
| --- | --- | --- |
| `main` | No alertó sobre `PeliculaController.uploadFile`. | Guarda el archivo recibido con su nombre original sin validar que sea una imagen: **carga arbitraria omitida por Semgrep**. |
| `practico-2` | Tampoco alertó sobre la carga. | El controlador valida formato, contenido, tamaño y dimensiones, reconstruye la imagen y le asigna un nombre nuevo: **se mitigó la causa observada en `main`**. |

En ambas ramas Semgrep señaló `Dockerfile` sin `USER` y aplicó al formulario Thymeleaf una regla CSRF de Django. El LLM identificó la incompatibilidad de esa regla, sin concluir que exista o falte protección CSRF real.

### Ejercicio 4 — Expresiones SpEL

| Rama | Semgrep | Revisión del LLM |
| --- | --- | --- |
| `main` | Alertó sobre `SpelEvaluator.java:14`. | `FuncionController.search` envía `buscar` al evaluador: **alerta válida para la búsqueda**. |
| `practico-2` | Conservó la misma alerta. | La búsqueda ya usa el texto directamente; **la alerta no demuestra que ese flujo siga siendo vulnerable**. La clase que evalúa expresiones continúa en el código y sería riesgosa si otro flujo le enviara datos no confiables. |

La alerta de `Dockerfile` sin `USER` aparece en las dos ramas. La diferencia principal es que la corrección cambió la llamada al evaluador, algo que no se refleja en el número de alertas SpEL.

### Ejercicio 5 — Contraseñas

| Rama | Semgrep | Revisión del LLM |
| --- | --- | --- |
| `main` | Emitió dos alertas por AES/ECB en `EncryptionService.java:50` y `:64`. | **Alertas válidas** para dos usos del mismo modo. `AuthController` almacena contraseñas mediante cifrado reversible y el servicio contiene una clave fija: **problemas que las alertas no describen**. |
| `practico-2` | Desaparecieron ambas alertas AES/ECB. | El servicio crea hashes Argon2id y el controlador verifica contraseñas sin descifrarlas: **la causa de las alertas AES/ECB ya no está**. |

En ambas ramas persiste la alerta de `Dockerfile` sin `USER` y Semgrep señala dos formularios Thymeleaf por ausencia de la etiqueta CSRF de Django. Estas últimas son falsos positivos de esa regla, no una evaluación de la protección CSRF de Spring.

## 6. Conclusión

La comparación por ejercicio muestra por qué **12 alertas en `main` y 11 en `practico-2` no equivalen a una diferencia de una sola vulnerabilidad**. Semgrep detectó AES/ECB, el evaluador SpEL y la configuración de los Dockerfile; también marcó casos que, al revisar su contexto, no representan la vulnerabilidad indicada. El LLM permitió seguir entradas hasta su uso y reconocer omisiones, especialmente el SQL del Ejercicio 2, que continúa en ambas ramas.

La rama `practico-2` corrigió las causas específicas del SQL del Ejercicio 1, el XSS del Ejercicio 2, la carga de archivos del Ejercicio 3, el recorrido SpEL del Ejercicio 4 y el cifrado reversible del Ejercicio 5. **Esto no significa que todos los ejercicios estén libres de riesgos:** permanecen el SQL del Ejercicio 2 y otras configuraciones señaladas. La revisión del LLM dependió de los archivos examinados y de las alertas conocidas; sus observaciones requieren verificación adicional cuando se quiera confirmar el comportamiento de la aplicación.

## Evidencias y reproducibilidad

Los resúmenes de terminal y los archivos `semgrep-main.json` y `semgrep-practico-2.json` respaldan las cifras de la sección 1. Los JSON se generaron fuera del repositorio; la siguiente tabla **transcribe sus 12 y 11 alertas** para que el informe se pueda leer sin abrir esos archivos. Las rutas son relativas a la carpeta de cada ejercicio. Un `0` significa que no hubo alerta de esa regla en esa rama, no que el código sea seguro.

| Ejercicio | Regla (nombre abreviado) | Archivo y línea | `main` | `practico-2` |
| --- | --- | --- | ---: | ---: |
| 1 | `missing-user` | `Dockerfile:12` | 1 | 1 |
| 1 | `sqlalchemy-execute-raw-query` | `app.py:44` | 0 | 1 |
| 2 | `missing-user` | `Dockerfile:12` | 1 | 1 |
| 2 | `direct-use-of-jinja2` | `test_seguridad.py:15` | 1 | 1 |
| 3 | `missing-user-entrypoint` | `Dockerfile:16` | 1 | 1 |
| 3 | `django-no-csrf-token` | `src/main/resources/templates/upload.html:39` | 1 | 1 |
| 4 | `missing-user-entrypoint` | `Dockerfile:12` | 1 | 1 |
| 4 | `spel-injection` | `src/main/java/com/cinebuscador/config/SpelEvaluator.java:14` | 1 | 1 |
| 5 | `missing-user-entrypoint` | `Dockerfile:12` | 1 | 1 |
| 5 | `use-of-aes-ecb` | `src/main/java/com/cinebuscador/config/EncryptionService.java:50, 64` | 2 | 0 |
| 5 | `django-no-csrf-token` | `src/main/resources/templates/index.html:45, 56` | 2 | 2 |
| **Total** | | | **12** | **11** |

En los dos JSON aparece una advertencia de análisis parcial de `Ejercicio2/templates/index.html`; por ello, las alertas no deben interpretarse como un examen completo de cada línea. La clasificación de falsos positivos y negativos se fundamenta en el código y se desarrolla por ejercicio en las secciones 4 y 5.
