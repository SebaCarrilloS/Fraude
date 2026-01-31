# Sistema de Detección de Fraude con Machine Learning

## 1. Contexto del problema

El fraude en transacciones financieras es un problema altamente desbalanceado: solo una pequeña fracción de las transacciones corresponde a fraude real, mientras que la gran mayoría son legítimas.

En este contexto, el objetivo **no es clasificar todas las transacciones**, sino **priorizar cuáles revisar**, ya que la revisión manual tiene un costo operativo y no es posible inspeccionar el 100% de los casos.

Por lo tanto, el problema se aborda como un **sistema de ranking de riesgo**, cuyo propósito es ordenar las transacciones desde las más a las menos riesgosas, habilitando políticas de revisión eficientes que maximicen la detección de fraude con recursos limitados.

---

## 2. Datos utilizados

Los datos utilizados en este proyecto provienen del dataset público **IEEE-CIS Fraud Detection** (Kaggle), el cual contiene información anonimizada de transacciones financieras y datos de identidad asociados.

El dataset se compone principalmente de dos tablas:

- `train_transaction.csv`: información de la transacción (montos, códigos, variables temporales).
- `train_identity.csv`: información adicional del usuario o dispositivo, no siempre disponible.

Por motivos de tamaño y licencia, los archivos **no se incluyen en el repositorio**.  
Para ejecutar el notebook localmente, se espera la siguiente estructura:

data/raw/
├── train_transaction.csv
└── train_identity.csv

## 3. Estrategia de validación

Para simular un escenario realista de producción y evitar *data leakage*, se utiliza un **split temporal** basado en la variable `TransactionDT`.

- El conjunto de entrenamiento contiene transacciones más antiguas.
- El conjunto de validación contiene transacciones posteriores en el tiempo.

Esta estrategia permite entrenar el modelo con información histórica y evaluarlo sobre datos “futuros”, replicando el comportamiento esperado en un entorno productivo.

---

## 4. Ingeniería de variables

Se construyen variables con foco en capturar **comportamiento anómalo**, más allá de información puntual de cada transacción:

- **Variables temporales**: hora del día e indicadores de transacciones nocturnas.
- **Missingness como señal**: número de valores faltantes por transacción, considerando que la ausencia de información no es aleatoria.
- **Agregaciones por grupo**: estadísticas de comportamiento por identificadores como tarjeta, dirección o dominio de correo, incluyendo:
  - conteo de transacciones,
  - monto promedio,
  - desviación estándar,
  - ratio entre el monto actual y el promedio histórico del grupo.

Estas variables permiten contextualizar cada transacción respecto a su comportamiento habitual.

---

## 5. Modelos evaluados

Se evaluaron dos enfoques principales:

- **Baseline**: Regresión Logística, utilizada como referencia inicial para validar la señal de las variables.
- **Modelo principal**: LightGBM, elegido por su capacidad para capturar relaciones no lineales y manejar grandes volúmenes de datos sin requerir escalamiento.

Dado el fuerte desbalance de clases, se aplican estrategias explícitas para manejar la clase minoritaria (fraude).

---

## 6. Evaluación del desempeño

El desempeño del modelo se evalúa utilizando métricas apropiadas para problemas desbalanceados:

- **PR-AUC (Precision–Recall AUC)** como métrica principal de ranking.
- **Precision@k y Recall@k**, que permiten interpretar el desempeño en términos operativos.

Estas métricas responden a preguntas del tipo:
- “Si reviso el top 1% de transacciones más riesgosas, ¿qué proporción es fraude?”
- “¿Qué porcentaje del fraude total logro capturar con esa política?”

---

## 7. Calibración de probabilidades

Aunque el modelo LightGBM presenta un excelente desempeño en ranking, sus scores no representan probabilidades calibradas.

Se evaluaron dos métodos de calibración:

- **Sigmoid (Platt scaling)**
- **Isotonic regression**

Los resultados muestran que la calibración Sigmoid mantiene el desempeño en ranking (PR-AUC), mientras que Isotonic introduce una leve degradación, por lo que se selecciona **Sigmoid** como calibrador final.

La calibración permite interpretar los scores como probabilidades de riesgo más realistas y es clave para una evaluación basada en costos.

---

## 8. Evaluación basada en costos

Para conectar el modelo con decisiones reales, se define una función de costo que considera:

- Un costo bajo asociado a la revisión manual de una transacción.
- Un costo significativamente mayor asociado a no detectar un fraude.

Se evalúan distintas políticas de revisión basadas en el porcentaje de transacciones revisadas (top-k).  
Los resultados muestran que:

- Políticas muy restrictivas presentan alta precisión, pero dejan pasar demasiado fraude.
- Revisar un mayor porcentaje reduce el fraude no detectado, aunque disminuye la precisión.
- El **mínimo costo total** se alcanza al revisar aproximadamente el **top 5%** de las transacciones más riesgosas.

---

## 9. Conclusiones

Este proyecto desarrolla un **sistema completo de detección de fraude**, que va más allá del entrenamiento de un modelo predictivo, integrando:

- validación temporal realista,
- ingeniería de variables basada en comportamiento,
- modelos robustos,
- calibración de probabilidades,
- y una política de decisión explícita basada en costos.

El resultado es un sistema **accionable**, capaz de apoyar decisiones operativas reales bajo restricciones de recursos, y alineado con prácticas utilizadas en entornos productivos de detección de fraude.

---

## Posibles extensiones

- Ajuste fino de hiperparámetros (Optuna).
- Evaluación con funciones de costo alternativas.
- Análisis de estabilidad temporal.
- Separación del pipeline en scripts productizables.