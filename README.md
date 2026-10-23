# Automatización del Ruteo de Bandejas Eléctricas en Entornos OpenBIM Mediante Optimización Jerárquica y Aprendizaje por Refuerzo Profundo

**Programa:** Maestría en Inteligencia Artificial  
**Curso:** Proyecto de Investigación 2 (Prof. Glen Rodríguez)  
**Institución:** Universidad Nacional de Ingeniería (UNI)  
**Autor:** Juan Villegas ([Juanvc07](https://github.com/Juanvc07))  
**Rama Activa:** `feature/semana4-feature-engineering`  
**Alcance del Sprint Actual:** EDA + Feature Engineering Tabular + Experimentos Controlados A/B sobre Variantes de los Datos (`cable_schedule.csv`)

---

## 1. Descripción del Proyecto y Planteamiento de la Problemática

En la industria de la construcción y el modelado BIM (*Building Information Modeling*), el diseño y la coordinación de las canalizaciones electromecánicas (MEP) para alimentadores principales se realiza tradicionalmente de manera **manual, fragmentada y reactiva**. Los proyectistas modelan las rutas de forma aislada y la detección de interferencias (*clash detection*) se efectúa tardíamente mediante software de coordinación (Navisworks o Solibri). Este flujo de trabajo genera decenas de colisiones espaciales que demandan semanas de rediseño manual y trasladan errores no resueltos a la etapa de ejecución en obra.

Esta problemática es especialmente crítica en **establecimientos de salud de alta complejidad (Categoría III-1)**, donde el pleno técnico disponible entre el cielo raso y las estructuras portantes presenta una saturación extrema. En estos corredores, las bandejas eléctricas de alimentadores principales deben convivir con ductos masivos de climatización (HVAC), vapor, gases medicinales y redes contraincendios por gravedad.

Las herramientas comerciales y los algoritmos convencionales de búsqueda de trayectorias (como $A^*$ elemental) no resuelven este escenario debido a deficiencias clave:
* **Inviabilidad Constructiva:** Generan ángulos de deflexión arbitrarios incompatibles con los accesorios comerciales prefabricados ($30^\circ$, $45^\circ$, $60^\circ$ y $90^\circ$).
* **Omisión de Restricciones Físicas y Normativas:** No consideran la rigidez mecánica ni el radio de curvatura admisible de cables de gran calibre ($500\text{ kcmil}$, $350\text{ kcmil}$), omiten la holgura superior libre de $300\text{ mm}$ exigida por la norma NEMA VE 2 para mantenimiento, e ignoran los factores de llenado normados por el Código Eléctrico Nacional (NEC Artículo 392).
* **Falta de Empaquetamiento Estratégico (*Bundling*):** Rutean cada circuito de forma independiente, dispersando canalizaciones en lugar de consolidar troncales compartidas que optimicen material y soportería.

Para resolver esta problemática, este proyecto de tesis propone un marco metodológico híbrido y desacoplado sobre estándar **IFC (ISO 16739)**, estructurado en dos etapas:
1. **Fase 1 (Sprint Actual - Caracterización Física y Feature Engineering Tabular):** Modelado de las restricciones electromecánicas, capacidad de llenado y rigidez de curvatura sobre el inventario real de alimentadores hospitalarios (`data/schedules/cable_schedule.csv`).
2. **Fase 2 (Próximos Sprints - Motor Espacial 3D e IA):** Extracción de mallas B-Rep con `IfcOpenShell`, generación del campo de distancias 3D ESDF y planificación cinemática con **Proximal Policy Optimization (PPO)**.

---

## 2. Hallazgos del EDA, Firma de Datos y Control de Colinealidad (VIF)

El análisis exploratorio ejecutado en [`notebooks/EDA_basico.ipynb`](notebooks/EDA_basico.ipynb) sobre los 24 circuitos alimentadores principales de un Hospital Categoría III-1 (conductores libres de halógenos **N2XOH 0.6/1 kV**) registró la siguiente firma estadística real:

| Variable Numérica | Media | Mediana (p50) | Percentil 90 (p90) | Varianza |
| :--- | :---: | :---: | :---: | :---: |
| `ternas` | $1.08$ | $1.00$ | $1.00$ | $0.08$ |
| `num_conductores` | $4.33$ | $4.00$ | $4.00$ | $1.28$ |
| `diametro_ext_mm` | $23.85\text{ mm}$ | $25.80\text{ mm}$ | $29.50\text{ mm}$ | $27.94$ |
| `peso_lineal_kg_m` | $1.74\text{ kg/m}$ | $1.98\text{ kg/m}$ | $2.58\text{ kg/m}$ | $0.49$ |
| `seccion_mm2` | $156.46\text{ mm}^2$ | $185.00\text{ mm}^2$ | $240.00\text{ mm}^2$ | $4346.69$ |
| `radio_min_curv_mm` | $286.25\text{ mm}$ | $310.00\text{ mm}$ | $354.00\text{ mm}$ | $4021.50$ |

### Decisiones Accionables del EDA:
1. **Restricción Crítica de Curvatura ($p90 = 354\text{ mm}$):** Los calibres de $350\text{ kcmil}$ y $500\text{ kcmil}$ superan el radio de giro de un accesorio comercial estándar ($300\text{ mm}$), justificando la creación del atributo `MS_curv_ref` y la bandera `flag_infraccion_curv`.
2. **Eliminación de Colinealidad mediante VIF:** El cálculo del Factor de Inflación de Varianza (VIF) en `notebooks/EDA_basico.ipynb` evidenció colinealidad severa en `seccion_mm2` ($\text{VIF} > 10$) respecto al diámetro y peso lineal. Se decidió **descartar `seccion_mm2`** en las variantes con Feature Engineering.
3. **Estabilización de Asimetría:** La carga mecánica total por metro lineal (`num_conductores * peso_lineal_kg_m`) presenta asimetría positiva en circuitos de doble terna, por lo que se aplicó la transformación `log1p` (`log_peso_total`).

---

## 3. Ingeniería de Atributos (Feature Engineering) y Selección con Evidencia

Cumpliendo la regla **«divide primero, transforma después»** (escalado `MinMaxScaler` ajustado estrictamente sobre el conjunto de entrenamiento en cada fold para garantizar **cero data leakage**), se definieron tres conjuntos progresivos centrados en los datos:

* **Set A (Baseline - Datos Crudos):** Variables originales sin transformar (`ternas`, `num_conductores`, `diametro_ext_mm`, `peso_lineal_kg_m`, `seccion_mm2`).
* **Set A+B (Variante 1 - Transformación y Codificación):**
  * Retiro de `seccion_mm2` por alta colinealidad (VIF).
  * Transformación logarítmica de carga: $\text{log\_peso\_total} = \ln(1 + N_{cond} \cdot W_{lineal})$.
  * Codificación *One-Hot* de subsistemas críticos hospitalarios (`es_emergencia`, `es_ups`).
* **Set A+B+C (Variante 2 - Ingeniería de Dominio Físico Normativo):**
  * **Ratio de Ocupación Normativa (`R_fill_ref`, NEC Art. 392):** Proporción del área transversal total de cobre respecto a la sección útil admisible (40%) de una bandeja de referencia de $300 \times 100\text{ mm}$:
    $$R_{fill\_ref} = \frac{N_{cond} \cdot \pi (D_{ext}/2)^2}{A_{util\_bandeja}}$$
  * **Margen de Seguridad de Curvatura (`MS_curv_ref`):** Relación continua entre el radio mínimo exigido por el conductor y el radio comercial de referencia ($300\text{ mm}$):
    $$MS_{curv\_ref} = 1.0 - \frac{R_{min}^{cable}}{300\text{ mm}}$$
  * **Flag de Infracción de Radio (`flag_infraccion_curv`):** Variable binaria que vale $1$ cuando $MS_{curv\_ref} < 0$.

### Evidencia de Importancia de Variables (¿Qué features se quedan?)
De acuerdo con el gráfico de **Importancia de Variables (Gain)** generado en la Celda 7 de [`notebooks/EDA_basico.ipynb`](notebooks/EDA_basico.ipynb):
1. **`R_fill_ref`** y **`log_peso_total`** concentran más del **80% de la capacidad predictiva** sobre el esfuerzo y requerimiento físico de canalización $\rightarrow$ **SE QUEDAN**.
2. **`MS_curv_ref`** y **`diametro_ext_mm`** aportan la discriminación de rigidez mecánica en calibres mayores $\rightarrow$ **SE QUEDAN**.
3. **`seccion_mm2`** $\rightarrow$ **SE DESCARTA** definitivamente por redundancia colineal verificada por VIF.

---

## 4. Diseño Experimental A/B sobre los Datos y Resultados Reales

Se evaluaron las 3 variantes sobre el dataset `data/schedules/cable_schedule.csv` empleando un esquema de **Validación Cruzada de 3 Folds (`KFold(n_splits=3, shuffle=True, random_state=42)`)** dentro de un `sklearn.pipeline.Pipeline` para impedir cualquier fuga de datos entre entrenamiento y validación:

### Tabla Comparativa de Resultados Reales (Generada en `logs/metrics_experimentos.csv`)

| ID Exp. | Variante de Datos (1 cambio por vez) | N° Features | Métrica Principal: RMSE (3-Fold CV) ↓ | Métrica Secundaria: MAE ↓ | Latencia Total (ms) ↓ | Archivo de Log |
| :--- | :--- | :---: | :---: | :---: | :---: | :--- |
| **EXP-00** | **Baseline (Set A: Datos Crudos)** | 5 | Ver `logs/metrics_baseline.txt` | Ver log | $\sim 280\text{ ms}$ | [`logs/metrics_baseline.txt`](logs/metrics_baseline.txt) |
| **EXP-01** | **Variante 1 (Set A+B: Transformados)** | 6 | Ver `logs/metrics_var1_fs1.txt` | Ver log | $\sim 285\text{ ms}$ | [`logs/metrics_var1_fs1.txt`](logs/metrics_var1_fs1.txt) |
| **EXP-02** | **Variante 2 (Set A+B+C: Full Dominio FE)** | 7 | **Menor RMSE y menor varianza entre folds** | **Menor MAE** | $\sim 290\text{ ms}$ | [`logs/metrics_var2_fs2.txt`](logs/metrics_var2_fs2.txt) |

*(Los valores exactos por fold, media, desviación estándar y timestamps generados al ejecutar el cuaderno se encuentran registrados automáticamente en [`logs/metrics_experimentos.csv`](logs/metrics_experimentos.csv) y en los gráficos de [`notebooks/EDA_basico.ipynb`](notebooks/EDA_basico.ipynb)).*

### Diagnóstico de la Curva de Aprendizaje (Celda 8 del Notebook)
* La curva de aprendizaje (*Train RMSE vs. Cross-Validation RMSE*) ejecutada en `notebooks/EDA_basico.ipynb` demuestra que la brecha entre entrenamiento y validación disminuye de forma sostenida conforme el tamaño de muestra pasa de 5 a 16 circuitos, convergiendo sin síntomas de sobreajuste severo (*overfitting*) gracias a la poda de profundidad (`max_depth=4`) y la eliminación de variables colineales.

---

## 5. Conclusiones del Sprint, Decisión y Próximos Pasos

1. **Decisión de Adopción:** Se **adopta la Variante 2 (`EXP-02`: Set A+B+C)** al reducir consistentemente el error medio (RMSE y MAE) y la desviación estándar entre los 3 folds, con un costo computacional prácticamente idéntico al Baseline.
2. **Riesgos y Próximos Pasos (Siguientes Sprints):**
   * **Riesgo a controlar:** Al integrar los modelos geométricos 3D (`.ifc`), el tamaño de los archivos y la densidad de obstáculos pueden incrementar la latencia de cómputo.
   * **Siguiente paso en el Roadmap:** Implementar el script de ingesta geométrica con `IfcOpenShell` sobre el modelo hospitalario para acoplar las variables tabulares de la Variante 2 (`R_fill_ref` y `MS_curv_ref`) con el margen espacial vertical ($MS_{clearance}$) en el entorno 3D.

---

## 6. Reproducibilidad y Estructura del Repositorio

### Instrucciones para Reproducir los Experimentos y Gráficos
```bash
# 1. Instalar librerías requeridas
py -m pip install pandas numpy matplotlib seaborn scikit-learn ipykernel

# 2. Abrir y ejecutar todas las celdas (1 a 8) del cuaderno interactivo:
# notebooks/EDA_basico.ipynb
