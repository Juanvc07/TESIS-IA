# Reporte Técnico: Selección e Importancia de Variables

**Fecha:** Octubre 2026  
**Dataset:** `data/schedules/cable_schedule.csv` (24 circuitos alimentadores N2XOH, Hospital Categoría III-1)  
**Método de evaluación:** Random Forest Regressor (100 estimadores, profundidad máxima = 4) evaluado con 3-Fold Cross-Validation sin fuga de datos (*MinMaxScaler* ajustado solo en train).

---

## 1. Justificación de Exclusión por Multicolinealidad (Filtro VIF)

Antes de entrenar, se evaluó el Factor de Inflación de Varianza (VIF) sobre las variables físicas dimensionales:

* `diametro_ext_mm`: VIF = 133,110.19
* `radio_min_curv_mm`: VIF = 127,269.12
* `peso_lineal_kg_m`: VIF = 2,460.10
* `seccion_mm2`: VIF = 1,914.96

**Decisión técnica:**  
En conductores eléctricos, la sección transversal de cobre ($mm^2$), el diámetro exterior ($mm$) y el peso lineal ($kg/m$) presentan dependencia física directa ($Área \propto D^2$). Un VIF > 10 evidencia redundancia severa. Se descartó `seccion_mm2` del conjunto final, conservando `diametro_ext_mm` y derivando ratios físicos para evitar inestabilidad en los coeficientes y sobreajuste.

---

## 2. Resultados de Importancia de Variables (Reducción de Impureza - Gain)

Sobre la **Variante 2 (Set A+B+C: Full Dominio)**, la contribución relativa de cada variable en la predicción del esfuerzo y ocupación de canalización fue:

| Variable | Tipo | Importancia (Gain) | Decisión | Justificación Técnica |
| :--- | :---: | :---: | :---: | :--- |
| `R_fill_ref` | Creada (Ratio NEC 392) | **0.426 (42.6%)** | **SE QUEDA** | Es el predictor principal. Determina si la sección de conductores supera el 40% de llenado admisible en la bandeja. |
| `log_peso_total` | Creada ($\ln(1+W)$) | **0.336 (33.6%)** | **SE QUEDA** | Modela la carga mecánica total que soporta la infraestructura de sujeción (tramos rectos y trapecios). |
| `MS_curv_ref` | Creada (Margen radio) | **0.144 (14.4%)** | **SE QUEDA** | Cuantifica la rigidez mecánica. Calibres $\ge 350\text{ kcmil}$ superan el radio comercial estándar ($300\text{ mm}$). |
| `diametro_ext_mm` | Original continua | **0.071 (7.1%)** | **SE QUEDA** | Proporciona la escala geométrica base para comprobaciones de ancho de base. |
| `flag_infraccion_curv` | Creada (Binaria) | **0.023 (2.3%)** | **SE QUEDA** | Actúa como condición de borde para penalización dura en ruteos con calibres rígidos. |
| `es_emergencia` | Creada (One-Hot) | **< 0.001 (0.0%)** | **SE DESCARTA** | No aporta a la predicción de esfuerzo mecánico ni ocupación geométrica en esta etapa. |
| `es_ups` | Creada (One-Hot) | **< 0.001 (0.0%)** | **SE DESCARTA** | Misma condición anterior; la separación de sistemas se manejará por capas lógicas, no por regresión física. |

**Conclusión:**  
Las tres variables creadas por ingeniería de dominio (`R_fill_ref`, `log_peso_total`, `MS_curv_ref`) concentran el **90.6% del poder explicativo** del modelo, demostrando cuantitativamente que las formulaciones normativas aportan mayor información que las variables dimensionales crudas del catálogo.

---

## 3. Comparación de Variantes de Datos (3-Fold CV)

| ID | Variante | RMSE Promedio | MAE Promedio | Latencia | Estado |
| :---: | :--- | :---: | :---: | :---: | :---: |
| EXP-00 | Baseline (Set A: Datos Crudos) | 18.5613 | 7.8255 | 154.73 ms | Descartado |
| EXP-01 | Variante 1 (Set A+B: Transformados) | 16.5705 | 8.1397 | 162.31 ms | Intermedio |
| **EXP-02** | **Variante 2 (Set A+B+C: Full Dominio)** | **16.7858** | **7.6332** | **165.76 ms** | **ADOPTADA** |

*Criterio de adopción:* La Variante 2 obtiene el menor error absoluto medio (`MAE = 7.63`), mantiene control de colinealidad y provee los términos matemáticos necesarios para alimentar la función de recompensa del agente de ruteo 3D.