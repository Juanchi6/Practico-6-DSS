# Práctico 6 — Análisis estático de código

**Materia:** Desarrollo de Software Seguro  
**Estado:** informe en elaboración. Las conclusiones se incorporarán tras revisar cada hallazgo.  
**Repositorio analizado:** aplicaciones de los Prácticos 2 y 4, en las ramas `main` y `practico-2`.

## 1. Objetivo y herramienta

El objetivo es analizar el código fuente de las aplicaciones usadas en los Prácticos 2 y 4, comparar la rama original (`main`) con la rama corregida (`practico-2`) y contrastar los resultados de una herramienta tradicional con una revisión del código realizada mediante un LLM.

Se eligió **Semgrep Community Edition (CE), versión 1.185.0**. Semgrep examina el código sin ejecutar las aplicaciones y aplica reglas para detectar patrones y flujos de datos potencialmente inseguros. En este trabajo se utilizó `--config auto`, que obtiene del registro las reglas comunitarias correspondientes al proyecto. Se registran los comandos, la versión, la cantidad de reglas y las salidas de ambos escaneos para poder reproducir la comparación.

La edición Community permite ejecutar reglas comunitarias y propias localmente. Frente a las capacidades comerciales, no incluye reglas Pro ni el análisis avanzado entre archivos y funciones de Semgrep Code. Tampoco incluye las funciones de Semgrep Supply Chain que aparecen como no activadas en el resumen de la CLI. Estas limitaciones ayudan a interpretar las vulnerabilidades que la configuración usada no detecta.

**Fuentes:** [Semgrep Community Edition](https://semgrep.dev/products/community-edition/) · [Comparación entre CE y la plataforma](https://semgrep.dev/products/semgrep-vs-ce/) · [Ejemplo oficial de `--config auto`](https://semgrep.dev/docs/semgrep-ci/sample-ci-configs).

## 2. Procedimiento y resultados de los escaneos

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

## 3. Comparación inicial de las ramas

Al comparar regla, archivo y línea en los JSON, **10 alertas aparecen en ambas ramas**. Hay **2 alertas exclusivas de `main`** y **1 exclusiva de `practico-2`**. Estas cifras explican la diferencia neta de 12 a 11; no implican por sí solas que solo haya cambiado una vulnerabilidad.

La identificación y clasificación de las alertas se completará tras revisar el código y el contexto de cada una.

## 4. Análisis de los hallazgos

**Pendiente de revisión.** Para cada alerta relevante se indicará la regla, archivo y línea, el comportamiento señalado y si corresponde a un problema real, un falso positivo o un caso que requiere más evidencia. Las alertas repetidas por una misma causa se explicarán juntas sin alterar la cantidad total informada por Semgrep.

Un **falso positivo** es una detección que no corresponde a la vulnerabilidad afirmada en el contexto del código. Un **falso negativo** es un problema presente y comprobado en el código que el escaneo no señaló. No se clasificarán hallazgos únicamente por el título de la regla.

## 5. Revisión estática con LLM

**Pendiente.** Se revisará el código fuente de ambas ramas con un modelo de lenguaje (LLM), registrando archivos, líneas, razonamiento y límites de cada observación. El LLM no ejecuta la aplicación ni sustituye la validación humana. Sus hallazgos se cotejarán con Semgrep y con la evidencia de los Prácticos 2 y 4.

## 6. Comparación y conclusión

**Pendiente.** Se explicará qué detectó cada método, qué omitió, qué alertas requirieron contexto y cómo cambiaron los resultados entre las ramas. La conclusión se redactará a partir de los hallazgos verificados.

## Evidencia para la defensa

Los dos resúmenes de terminal y los archivos `semgrep-main.json` y `semgrep-practico-2.json` respaldan las cifras. El informe incorporará una tabla breve con los casos importantes para que pueda presentarse y explicarse sin recorrer todos los registros JSON.
