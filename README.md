# Automatización del Ruteo de Bandejas Eléctricas en Entornos OpenBIM Mediante Optimización Jerárquica y Aprendizaje por Refuerzo Profundo

**Programa:** Maestría en Inteligencia Artificial

**Institución:** Universidad Nacional de Ingeniería (UNI)

**Autor:** Juan Villegas

**Línea de Investigación:** Inteligencia Artificial Aplicada, Deep Reinforcement Learning, Geometría Computacional y OpenBIM

---

## Descripción del Proyecto

El modelado y la coordinación espacial de canalizaciones eléctricas para alimentadores principales en proyectos de gran envergadura constituye un proceso altamente iterativo y susceptible a errores geométricos. Convencionalmente, la resolución de interferencias (clashes) y el cumplimiento de normativas de constructabilidad física dependen de revisiones manuales tardías, lo que genera sobrecostos y retrasos en obra.

Este proyecto de tesis propone e implementa un marco metodológico híbrido, jerárquico y desacoplado para la automatización del ruteo 3D de bandejas portacables sobre modelos federados en formato estándar **IFC (ISO 16739)**. El sistema desacopla el cálculo algorítmico del software de modelado comercial mediante una representación continua del espacio, resolviendo de forma articulada el macro-ruteo de troncales con agrupamiento (bundling) y la micro-planificación cinemática local de accesorios mediante **Aprendizaje por Refuerzo Profundo (Deep Reinforcement Learning - PPO)**.

---

## Arquitectura Metodológica del Sistema

El pipeline computacional se compone de cinco fases secuenciales desacopladas:

1. **Ingesta Semántica y Representación Continua (Nivel 1):**
* Extracción de mallas trianguladas exactas (B-Rep) de elementos estructurales (`IfcBeam`, `IfcColumn`, `IfcSlab`) y redes mecánicas preexistentes mediante `IfcOpenShell`.
* Voxelización semántica en tensores dispersos en GPU para optimizar el consumo de memoria.
* Generación de un Campo de Distancias con Signo Tridimensional (*3D Euclidean Signed Distance Field* - ESDF), permitiendo consultas de colisión y gálibo en tiempo constante $O(1)$.


2. **Macro-Ruteo Global y Empaquetamiento de Circuitos (Nivel 2):**
* Abstracción del pleno técnico del edificio en un grafo tridimensional conexo $G = (V, E)$.
* Formulación y resolución del problema combinatorio de agrupamiento (*Cable Harness Routing Problem* - CHRP) para balancear la longitud total de conductores de cobre y la apertura de canalizaciones compartidas.
* Dimensionamiento automático de anchos de bandeja según los límites de ocupación del Código Eléctrico Nacional (NEC Artículo 392).


3. **Micro-Optimización Cinemática Guiada por IA (Nivel 3):**
* Formulación de un Proceso de Decisión de Markov (MDP) en un entorno Gymnasium.
* Agente Actor-Crítico entrenado con **Proximal Policy Optimization (PPO)** con espacio de acciones discretizado a catálogo prefabricado ($30^\circ$, $45^\circ$, $60^\circ$, $90^\circ$).
* Enmascaramiento de acciones (*action masking*) para garantizar el respeto al radio de curvatura admisible mínimo ($R_{min}^{cable}$) del alimentador de mayor calibre.
* Evaluación de espacios libres de mantenimiento ($300\text{ mm}$ superiores según norma NEMA VE 2) contra el campo ESDF.


4. **Escritura y Generación Nativa OpenBIM (Nivel 4):**
* Generación de tramos prismáticos `IfcCableCarrierSegment` y accesorios normalizados `IfcCableCarrierFitting` en el archivo IFC final.
* Interconexión lógica de puertos mediante `IfcDistributionPort` y asignación al sistema `IfcDistributionSystem`.
* Inyección de metadatos de ingeniería (factor de llenado, peso lineal, código de circuito) en conjuntos de propiedades estandarizados (`IfcPropertySet`).


5. **Auditoría y Validación:**
* Evaluación de interferencias y reglas constructivas en verificadores de modelos IFC.



---

## Estructura del Repositorio

* `configs/`: Archivos de configuración para hiperparámetros de entrenamiento PPO y tablas de catálogo normativo (NEC/NEMA).
* `data/`: Modelos IFC federados de prueba (`raw/`), tablas de alimentadores (`schedules/`) y modelos IFC generados (`outputs/`).
* `docs/`: Artículos científicos de referencia (`papers/`), borrador de la tesis (`thesis/`) y fichas normativas (`standards/`).
* `logs/`: Registros de entrenamiento, gráficos de recompensa acumulada de TensorBoard y telemetría de colisiones.
* `notebooks/`: Cuadernos interactivos para inspección de geometría IFC, visualización de tensores y pruebas preliminares.
* `slides/`: Diapositivas de avance para las asesorías de tesis y comités de posgrado.
* `src/`: Código fuente modular en Python (ingesta, entorno Gymnasium, agentes PPO, optimización en grafos y exportación IFC).
* `requirements.txt`: Lista de dependencias de Python requeridas.

---

## Resultados Esperados

* **Validación Exploratoria Inicial:**
* Visualización y comprobación de extracción B-Rep exacta de elementos IFC en `notebooks/01_extraccion_ifcopenshell.ipynb`.


* **Línea Base (Baseline):**
* Implementación de búsqueda heurística determinista ($A^*$ tramo por tramo sin agrupamiento unificado) con métricas registradas en `logs/metrics_baseline.txt`.


* **Pipeline Jerárquico Propuesto:**
* Inferencia de la arquitectura acoplada (CHRP + PPO cinemático) con métricas en `logs/metrics_hierarchical.txt`.


* **Métricas Principales de Desempeño:**
* **Tasa de Colisiones (Clashes):** 0 colisiones duras contra elementos estructurales portantes.
* **Constructabilidad Física:** 100% de accesorios correspondientes a ángulos discretos de catálogo ($30^\circ, 45^\circ, 60^\circ, 90^\circ$).
* **Cumplimiento de Curvatura:** 0 violaciones del radio admisible de flexión del cable.
* **Factor de Agrupamiento:** Reducción porcentual en metros lineales de bandeja frente a trazados individuales.



---

## Roadmap de Desarrollo

* [x] **Semana 1-2:** Definición del marco metodológico, diseño del pipeline reproducible, estructuración del repositorio y análisis de riesgos computacionales.
* [ ] **Semana 3-4:** Ingesta semántica mediante `IfcOpenShell`, extracción B-Rep e indexación de obstáculos en tensores dispersos.
* [ ] **Semana 5-6:** Implementación del campo de distancias continuo (ESDF) y construcción del grafo de navegación en pasillos técnicos.
* [ ] **Semana 7-8:** Formulación y resolución del macro-ruteo y agrupamiento de alimentadores (CHRP).
* [ ] **Semana 9-10:** Configuración del entorno Gymnasium, definición de recompensas normativas y entrenamiento del agente PPO cinemático.
* [ ] **Semana 11-12:** Módulo de exportación y serialización nativa de entidades `IfcCableCarrierSegment` y `IfcCableCarrierFitting` en IFC.
* [ ] **Semana 13-14:** Auditoría experimental de interferencias, consolidación de métricas comparativas y redacción del informe final de tesis.

---

## Licencia

Uso académico y de investigación – Maestría en Inteligencia Artificial – Universidad Nacional de Ingeniería (UNI). Todos los derechos reservados.
