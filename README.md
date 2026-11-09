# Predicción de Reincidencia Penal con Árboles de Decisión (CART) y Justicia Predictiva

Proyecto de clasificación supervisada aplicado al dataset de libertad condicional de Pensilvania (Parole). El objetivo fue modelar el riesgo de violación de la condicional o reincidencia penal, priorizando la detección de casos de riesgo (Recall / Sensibilidad) sobre una base fuertemente desbalanceada.

---

## 1. Arquitectura del Flujo y Pipeline

Para evitar cualquier fuga de datos (data leakage), el 20% de los datos se separó al inicio como conjunto de prueba (Test). La codificación One-Hot, el balanceo por sobremuestreo y el ajuste del árbol se ejecutaron dentro de cada partición de validación cruzada:

<div align="center">
  <img src="pipeline_arbol.png" alt="Pipeline de Modelado Árbol CART" width="800">
</div>

---

## 2. Limpieza y Tratamiento de Datos

* **Análisis de duplicados:** Se identificaron 48 filas con valores idénticos en todas las columnas. Al no contar con un identificador único por individuo y ser variables discretas con pocas categorías posibles, se conservaron para evitar la eliminación artificial de perfiles reales y no alterar la proporción de clases.
* **Manejo de valores nulos:** El conjunto no presentó valores faltantes, por lo que no se requirió imputación.
* **Valores atípicos:** Los árboles de decisión son modelos no paramétricos basados en particiones de espacio ortogonales (umbrales), por lo que son naturalmente robustos ante valores extremos y no requieren tratamiento de outliers.
* **Desbalance severo:** La clase de interés (Status = 1, reincidencia) representaba únicamente el 11.6% del total (78 de 675 casos), exigiendo estrategias dedicadas de remuestreo.

---

## 3. Comparativa de Balanceo y Optimización (GridSearchCV)

Se evaluaron 9 estrategias de balanceo mediante GridSearchCV con 5-Fold Stratified Cross-Validation, explorando hiperparámetros de pre-poda:

* **Criterio de división (criterion):** Gini vs. Entropy
* **Profundidad máxima (max_depth):** [2, 3, 4, 5, 6, 8, 10, None]
* **Mínimo de muestras para división (min_samples_split):** [2, 5, 10, 20]
* **Mínimo de muestras por hoja (min_samples_leaf):** [1, 2, 5, 10]
* **Variables por división (max_features):** Todas (None) vs. Raíz cuadrada (sqrt)

### Configuración del Mejor Modelo
* **Estrategia de balance seleccionada:** Random Oversampling (la de mayor Recall y menor desviación estándar entre folds)
* **Criterio de impureza:** Gini
* **Profundidad del árbol:** 3
* **Número de hojas:** 8
* **Post-poda por costo-complejidad:** El algoritmo seleccionó un valor de ccp_alpha = 0, confirmando que la pre-poda con profundidad 3 ya evitaba el sobreajuste sin requerir podar ramas adicionales.
* **Recall en Validación Cruzada:** 0.8397 (+- 0.0881)
* **F1-Score en Validación Cruzada:** 0.3473
* **ROC-AUC en Validación Cruzada:** 0.8200

---

## 4. Estructura y Reglas del Árbol Final

El modelo generado es compacto e interpretable, facilitando la auditoría de cada regla de decisión:

<div align="center">
  <img src="arbol_decision.png" alt="Estructura del Árbol de Decisión Final" width="800">
</div>

### Importancia de Variables y Reglas Principales
1. **Estado_Virginia (59.77% de importancia):** Es la división raíz del árbol. Fuera de Virginia se clasifica como riesgo de reincidencia en la mayoría de los perfiles, salvo casos sin múltiples ofensas previas y con delitos de tránsito.
2. **Ofensas_No (22.91% de importancia):** La ausencia de múltiples condenas previas actúa como un fuerte reductor del riesgo estimado.
3. **Crimen_Transito (6.20%), Sentencia (3.34%) y Tiempo (3.31%):** Modulan las predicciones en ramas específicas.
4. **Atributos sensibles:** Género no fue utilizado en las divisiones, mientras que Raza aparece en un corte pero ambas ramas asignan la misma clase final (1), evidenciando que no altera la decisión final del clasificador.

---

## 5. Resultados en el Conjunto de Test

Evaluación final sobre el 20% de datos apartados (135 casos de prueba):

* **Recall (Sensibilidad):** 81.25% (se identificaron 13 de las 16 reincidencias reales)
* **Precision:** 20.63%
* **F1-Score:** 32.91%
* **Accuracy:** 60.74%
* **ROC-AUC:** 0.7623

### Matriz de Confusión

<div align="center">
  <img src="matriz_confusion.png" alt="Matriz de Confusión Árbol CART" width="480">
</div>

* **Verdaderos Negativos (TN):** 69 personas sin reincidencia clasificadas correctamente.
* **Falsos Positivos (FP):** 50 personas que cumplieron su condicional pero fueron marcadas como riesgo potencial.
* **Falsos Negativos (FN):** 3 casos de reincidencia no detectados.
* **Verdaderos Positivos (TP):** 13 reincidencias detectadas oportunamente.

---

## 6. Consideraciones Éticas en Justicia Predictiva

Al priorizar el Recall para minimizar el riesgo público, el modelo presenta una precisión de 20.63%, lo que significa que aproximadamente 4 de cada 5 personas señaladas como riesgo habrían cumplido su libertad condicional sin infracciones. Asimismo, la fuerte dependencia de la variable jurisdiccional (Estado) refleja disparidades en las tasas de arresto o supervisión local antes que factores criminológicos individuales. Por estas razones, este tipo de herramientas debe concebirse como apoyo para asignar recursos de supervisión y rehabilitación, nunca como criterio automatizado determinante para revocar o denegar derechos de libertad.
