# Optimización del Ruteo de Bandejas Eléctricas en Entornos OpenBIM mediante Voxelización Semántica, Búsqueda Heurística y Aprendizaje por Refuerzo Profundo

**Programa:** Maestría en Inteligencia Artificial  
**Curso:** Proyecto de Investigación II (Dr. Glen Darío Rodríguez Rafael)  
**Institución:** Universidad Nacional de Ingeniería (UNI) – Unidad de Posgrado FIIS  
**Tesista:** Ing. Juan José Villegas Calvay ([Juanvc07](https://github.com/Juanvc07))  
**Asesor:** María Fernanda Tejada Begazo  
**Entregable Consolidado:** [`docs/thesis/Informe_Parcial_PI2_Plan_Tesis_Juan_Villegas.docx`](docs/thesis/Informe_Parcial_PI2_Plan_Tesis_Juan_Villegas.docx) | [`notebooks/Plan_Tesis_y_Protocolo_PI2.ipynb`](notebooks/Plan_Tesis_y_Protocolo_PI2.ipynb)

---

## 1. Descripción del Proyecto y Planteamiento de la Problemática

En la industria de la construcción y el modelado BIM (*Building Information Modeling*), el diseño y la coordinación espacial de las **bandejas eléctricas portacables** para alimentadores principales se realiza tradicionalmente de manera manual, fragmentada y reactiva. En **establecimientos de salud de alta complejidad (Hospitales Categoría III-1)**, los plenos técnicos entre el cielo raso y las estructuras portantes presentan una saturación extrema debido a la convergencia de ductos HVAC, vapor, gases medicinales, redes contraincendios y la segregación obligatoria entre subsistemas eléctricos (Normal, Emergencia y UPS).

### Arquitectura Híbrida Propuesta (2 Fases)
1. **Fase 1 (Completada — Caracterización Físico-Normativa Tabular):** Modelado de ocupación en bandeja (`R_fill_ref`), carga mecánica (`log_peso_total`) y margen de curvatura (`MS_curv_ref`) sobre el inventario real de 24 alimentadores hospitalarios (`data/schedules/cable_schedule.csv`), con control de colinealidad (VIF) y validación cruzada sin fuga de datos (*zero data leakage*).
2. **Fase 2 (En Desarrollo — Motor Espacial OpenBIM + IA):** Extracción geométrica B-Rep desde IFC 4 mediante `IfcOpenShell`, voxelización semántica (50 mm) con Campos de Distancia Signada Euclidianos (**ESDF**), guía macro-heurística **A* 3D** y planificación secuencial con **Proximal Policy Optimization (PPO)** dotado de enmascaramiento cinemático de acciones (*Action Masking*).

---

## 2. Resultados Reales de Experimentos A/B (3-Fold Cross-Validation)

| ID Exp. | Variante de Datos (1 cambio por vez) | Features | RMSE (3-Fold CV) | MAE Medio | Latencia | Decisión |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: |
| **EXP-00** | Baseline (Set A: Datos Crudos) | 5 | 18.5613 ± 12.2989 | 7.8255 | 154.73 ms | Descartado |
| **EXP-01** | Variante 1 (Set A+B: Transformados + Filtro VIF) | 6 | 16.5705 ± 15.9042 | 8.1397 | 162.31 ms | Intermedio |
| **EXP-02** | **Variante 2 (Set A+B+C: Full Dominio FE)** | 7 | **16.7858 ± 17.6488** | **7.6332** | **165.76 ms** | **ADOPTADA** |

---

## 3. Documentación del Plan de Tesis y Entregables

* **Plan de Tesis Oficial (Word):** [`docs/thesis/Informe_Parcial_PI2_Plan_Tesis_Juan_Villegas.docx`](docs/thesis/Informe_Parcial_PI2_Plan_Tesis_Juan_Villegas.docx)
* **Cuaderno Ejecutivo del Plan de Tesis:** [`notebooks/Plan_Tesis_y_Protocolo_PI2.ipynb`](notebooks/Plan_Tesis_y_Protocolo_PI2.ipynb)
* **Matriz de Consistencia Metodológica:** [`docs/matriz_consistencia.md`](docs/matriz_consistencia.md)
* **Reporte Técnico de Importancia de Variables y VIF:** [`docs/reporte_importancia_variables.md`](docs/reporte_importancia_variables.md)
* **Cuaderno de EDA y Ablación Experimental:** [`notebooks/EDA_basico.ipynb`](notebooks/EDA_basico.ipynb)
* **Estado del Arte (13 artículos y tesis base):** [`docs/papers/`](docs/papers/)
