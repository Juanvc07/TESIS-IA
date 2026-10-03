# Automatización del Ruteo de Bandejas Eléctricas en Entornos OpenBIM Mediante Optimización Jerárquica y Aprendizaje por Refuerzo Profundo

**Programa:** Maestría en Inteligencia Artificial

**Institución:** Universidad Nacional de Ingeniería (UNI)

**Autor:** Juan Villegas

**Línea de Investigación:** Inteligencia Artificial Aplicada, Deep Reinforcement Learning, Geometría Computacional y OpenBIM

---

## Descripción del Proyecto

El modelado y la coordinación espacial de canalizaciones eléctricas para alimentadores principales en proyectos de gran envergadura constituye un proceso altamente iterativo y susceptible a errores geométricos. Convencionalmente, la resolución de interferencias (*clashes*) y el cumplimiento de normativas de constructabilidad física dependen de revisiones manuales tardías, lo que genera sobrecostos y retrasos en obra.

Este proyecto de tesis propone e implementa un marco metodológico híbrido, jerárquico y desacoplado para la automatización del ruteo 3D de bandejas portacables sobre modelos federados en formato estándar **IFC (ISO 16739)**. El sistema desacopla el cálculo algorítmico del software de modelado comercial mediante una representación continua del espacio, resolviendo de forma articulada el macro-ruteo de troncales con agrupamiento (*bundling*) y la micro-planificación cinemática local de accesorios mediante **Aprendizaje por Refuerzo Profundo (Deep Reinforcement Learning - PPO)**.

---

## Casos de Estudio y Obtención de Datos

### 1. Casos de Estudio: Infraestructura Hospitalaria Compleja (Categoría III-1)

Para el entrenamiento, calibración y validación experimental de los algoritmos se cuenta con **dos modelos BIM federados de proyectos hospitalarios reales clasificados en Categoría III-1** (establecimientos de salud de alta complejidad con servicios de hospitalización, unidades de cuidados intensivos, centros quirúrgicos y diagnóstico especializado).

La elección de esta tipología responde a su alta exigencia técnica:

* **Congestión del Pleno Técnico:** Alta densidad de redes mecánicas (HVAC), vapor, gases medicinales y redes contraincendios que compiten por el espacio libre bajo losas y vigas.
* **Jerarquía y Continuidad Eléctrica:** Presencia obligatoria de múltiples sistemas de suministro (red normal, emergencia por grupos electrógenos y sistemas ininterrumpidos UPS) con requerimientos críticos de segregación física de alimentadores principales.
* **Geometría No Homogénea:** Diversidad de vanos, placas estructurales de cortante y vigas peraltadas que representan retos de constructabilidad para el paso de canalizaciones.

### 2. Metodología de Obtención y Preprocesamiento de la Data

La obtención y estructuración de la información no depende de bases de datos externas sintéticas, sino de la extracción directa desde los modelos digitales de ingeniería:

* **Exportación Federada OpenBIM (IFC4):** Los modelos de Arquitectura, Estructuras y redes MEP preexistentes se exportan en formato estándar IFC4 (*Reference View* con teselación B-Rep exacta), garantizando la geometría real de vanos y elementos sin sobrestimación por envolventes cúbicas.
* **Extracción Semántica Automatizada con `IfcOpenShell`:** Mediante scripts en Python se parsea el archivo IFC extrayendo las mallas y propiedades de:
* Elementos estructurales no franqueables (`IfcBeam`, `IfcColumn`, `IfcSlab`).
* Tabiquería y cerramientos condicionados (`IfcWall`, evaluando el parámetro `LoadBearing`).
* Redes existentes (`IfcDuctSegment`, `IfcPipeSegment`) para calcular volúmenes de gálibo normativo.
* Zonas de tránsito y plenos técnicos (`IfcSpace`) para definir los corredores válidos de ruteo.


* **Tablas de Alimentadores Eléctricos (*Cable Schedules*):** Se estructura la demanda eléctrica en archivos tabulares (`.json` / `.csv`) que contienen la relación formal de alimentadores principales del hospital: punto de partida (subestación / tableros generales), tableros secundarios de distribución, número de conductores por fase, calibre comercial (AWG/kcmil), diámetro exterior y peso por metro lineal.
* **Representación Espacial Continua:** La geometría extraída se procesa en memoria mediante tensores dispersos (coordenadas COO en GPU) y se transforma en un Campo de Distancias con Signo Tridimensional (3D ESDF), permitiendo al agente de optimización verificar holguras e interferencias en tiempo constante $O(1)$.

---

## Arquitectura Metodológica del Sistema

El pipeline computacional se compone de cinco fases secuenciales desacopladas:

1. **Ingesta Semántica y Representación Continua (Nivel 1):**
* Extracción de mallas trianguladas exactas (B-Rep) con `IfcOpenShell`.
* Voxelización semántica en tensores dispersos en GPU para optimizar el consumo de memoria.
* Generación del campo de distancias continuo (3D ESDF) para consultas inmediatas de proximidad a obstáculos.


2. **Macro-Ruteo Global y Empaquetamiento de Circuitos (Nivel 2):**
* Abstracción de los pasillos técnicos del hospital en un grafo tridimensional conexo $G = (V, E)$.
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
* Evaluación de interferencias y reglas constructivas en verificadores de modelos IFC (Solibri Office / scripts de auditoría).



---

## Estructura del Repositorio

* `configs/`: Archivos de configuración para hiperparámetros de entrenamiento PPO y tablas de catálogo normativo (NEC/NEMA).
* `data/`:
* `data/raw/`: Modelos IFC federados de los hospitales Categoría III-1.
* `data/schedules/`: Tablas de alimentadores y circuitos principales (*Cable Schedules*).
* `data/processed/`: Tensores dispersos y representaciones espaciales optimizadas.
* `data/outputs/`: Modelos IFC finales enriquecidos con las canalizaciones generadas.


* `docs/`: Artículos científicos de referencia (`papers/`), borrador de la tesis (`thesis/`) y fichas técnicas (`standards/`).
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
* **Factor de Agrupamiento:** Reducción porcentual en metros lineales de canalización frente a trazados individuales.



---

## Roadmap de Desarrollo

* [x] **Semana 1-2:** Definición del marco metodológico, estructuración del repositorio y recopilación de modelos hospitalarios Categoría III-1.
* [ ] **Semana 3-4:** Ingesta semántica mediante `IfcOpenShell`, extracción B-Rep e indexación de obstáculos en tensores dispersos.
* [ ] **Semana 5-6:** Implementación del campo de distancias continuo (ESDF) y construcción del grafo de navegación en pasillos técnicos.
* [ ] **Semana 7-8:** Formulación y resolución del macro-ruteo y agrupamiento de alimentadores (CHRP).
* [ ] **Semana 9-10:** Configuración del entorno Gymnasium, definición de recompensas normativas y entrenamiento del agente PPO cinemático.
* [ ] **Semana 11-12:** Módulo de exportación y serialización nativa de entidades `IfcCableCarrierSegment` y `IfcCableCarrierFitting` en IFC.
* [ ] **Semana 13-14:** Auditoría experimental de interferencias, consolidación de métricas comparativas y redacción del informe final de tesis.

---

## Licencia

Uso académico y de investigación – Maestría en Inteligencia Artificial – Universidad Nacional de Ingeniería (UNI). Todos los derechos reservados.
