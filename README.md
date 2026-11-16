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

**Para defenderlo:** Semgrep no detectó el SQL vulnerable de `main` y sí marcó la versión parametrizada. Por eso revisamos cómo llega la entrada del usuario a `db.execute()`, en vez de aceptar la alerta sin contexto.

**Otros hallazgos:** pendientes de validar y documentar.

## 5. Revisión estática con LLM

**Pendiente.** Se revisará el código fuente de ambas ramas con un modelo de lenguaje (LLM), registrando archivos, líneas, razonamiento y límites de cada observación. El LLM no ejecuta la aplicación ni sustituye la validación humana. Sus hallazgos se cotejarán con Semgrep y con la evidencia de los Prácticos 2 y 4.

## 6. Comparación y conclusión

**Pendiente.** Se explicará qué detectó cada método, qué omitió, qué alertas requirieron contexto y cómo cambiaron los resultados entre las ramas. La conclusión se redactará a partir de los hallazgos verificados.

## Evidencia para la defensa

Los dos resúmenes de terminal y los archivos `semgrep-main.json` y `semgrep-practico-2.json` respaldan las cifras. El informe incorporará una tabla breve con los casos importantes para que pueda presentarse y explicarse sin recorrer todos los registros JSON.
