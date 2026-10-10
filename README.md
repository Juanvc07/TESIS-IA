# Automatización del Ruteo de Bandejas Eléctricas en Entornos OpenBIM Mediante Optimización Jerárquica y Aprendizaje por Refuerzo Profundo

**Programa:** Maestría en Inteligencia Artificial

**Curso:** Proyecto de Investigación 2

**Institución:** Universidad Nacional de Ingeniería (UNI)

**Autor:** Juan Villegas ([Juanvc07](https://www.google.com/search?q=https://github.com/Juanvc07))

**Rama del Repositorio:** `feature/semana4-feature-engineering`

**Línea de Investigación:** Inteligencia Artificial Aplicada, Deep Reinforcement Learning, Feature Engineering en Optimización y OpenBIM[cite: 1, 262]

---

## 1. Descripción del Proyecto y Planteamiento de la Problemática

En la industria de la construcción y el modelado BIM (*Building Information Modeling*), el diseño y la coordinación de las canalizaciones electromecánicas (MEP) para alimentadores principales se realiza tradicionalmente de manera **manual, fragmentada y reactiva**[cite: 1, 20, 29]. Los proyectistas modelan las rutas de forma aislada y la detección de interferencias (*clash detection*) se efectúa tardíamente mediante software de coordinación (Navisworks o Solibri)[cite: 1, 30]. Este flujo de trabajo genera decenas de colisiones espaciales que demandan semanas de rediseño manual y trasladan errores no resueltos a la etapa de ejecución en obra[cite: 1, 29].

Esta problemática es especialmente crítica en **establecimientos de salud de alta complejidad (Categoría III-1)**, donde el pleno técnico disponible entre el cielo raso y las estructuras portantes presenta una saturación extrema[cite: 5]. En estos corredores, las bandejas eléctricas de alimentadores principales deben convivir con ductos masivos de climatización (HVAC), vapor, gases medicinales y redes contraincendios por gravedad[cite: 1, 112].

Las herramientas comerciales y los algoritmos convencionales de búsqueda de trayectorias (como $A^*$ elemental) no resuelven este escenario debido a deficiencias clave[cite: 1, 16]:

* **Inviabilidad Constructiva:** Generan ángulos de deflexión arbitrarios incompatibles con los accesorios comerciales prefabricados ($30^\circ$, $45^\circ$, $60^\circ$ y $90^\circ$)[cite: 1].
* **Omisión de Restricciones Físicas y Normativas:** No consideran la rigidez mecánica ni el radio de curvatura admisible de cables de gran calibre ($500\text{ kcmil}$, $250\text{ kcmil}$), omiten la holgura superior libre de 300 mm exigida por la norma NEMA VE 2 para mantenimiento, e ignoran los factores de llenado normados por el Código Eléctrico Nacional (NEC Artículo 392)[cite: 1, 168, 182, 224].
* **Falta de Empaquetamiento Estratégico (*Bundling*):** Rutean cada circuito de forma independiente, dispersando canalizaciones en lugar de consolidar troncales compartidas que optimicen material y soportería[cite: 1, 131, 133].
* **Sobrecarga Computacional:** Los intentos previos dentro de APIs propietarias (como Autodesk Revit) colapsan por latencia de entrada/salida y por la sobreestimación geométrica de cajas delimitadoras (*Bounding Boxes*)[cite: 1, 85].

Para resolver esta problemática, este proyecto de tesis propone e implementa un marco metodológico híbrido, jerárquico y desacoplado sobre modelos en formato estándar **IFC (ISO 16739)**[cite: 1, 40]. El sistema desacopla el cálculo del software comercial hacia un motor continuo en Python/GPU, combinando optimización combinatoria en grafos para el macro-ruteo de troncales con **Aprendizaje por Refuerzo Profundo (Deep Reinforcement Learning - PPO)** para la micro-planificación cinemática de accesorios[cite: 1, 2, 50, 133].

---

## 2. Casos de Estudio y Obtención de Datos

### Casos de Estudio: Infraestructura Hospitalaria Compleja (Categoría III-1)

Para el entrenamiento, calibración y validación experimental del sistema se cuenta con **dos modelos BIM federados de proyectos hospitalarios reales clasificados en Categoría III-1** (establecimientos con unidades de cuidados intensivos, centros quirúrgicos, hospitalización y diagnóstico especializado)[cite: 5].

Estos casos permiten someter el algoritmo a condiciones de máxima exigencia geométrica, alta densidad de redes mecánicas concurrentes y requerimientos estrictos de segregación física de alimentadores (red comercial, grupos electrógenos de emergencia y sistemas ininterrumpidos UPS)[cite: 1, 5, 112].

### Metodología de Obtención y Preprocesamiento de la Data

La estructuración del entorno computacional no depende de bases de datos externas sintéticas, sino de la extracción directa de la física del edificio[cite: 1]:

* **Exportación Federada OpenBIM (IFC4):** Los modelos de Arquitectura, Estructuras y redes MEP preexistentes se exportan en formato estándar IFC4 (*Reference View* con teselación B-Rep exacta), garantizando la geometría real de vanos y vigas sin cajas delimitadoras artificiales[cite: 1, 41].
* **Extracción Semántica Automatizada con `IfcOpenShell`:** Mediante scripts en Python se parsea el archivo IFC extrayendo las mallas y propiedades de elementos estructurales (`IfcBeam`, `IfcColumn`, `IfcSlab`), tabiquería ligera (`IfcWall`, evaluando el parámetro `LoadBearing`), redes mecánicas (`IfcDuctSegment`, `IfcPipeSegment`) y plenos técnicos (`IfcSpace`)[cite: 2, 69, 74].
* **Tablas de Alimentadores Eléctricos (*Cable Schedules*):** La demanda eléctrica se define en archivos tabulares (`.json` / `.csv`) que contienen el inventario formal de circuitos: origen, destino, cantidad de conductores, calibre comercial (AWG/kcmil), diámetro exterior y peso lineal[cite: 68].
* **Representación Espacial Continua:** La geometría extraída se indexa en memoria mediante tensores dispersos en GPU y se transforma en un Campo de Distancias con Signo Tridimensional (3D ESDF), permitiendo consultar colisiones y distancias de seguridad en tiempo constante $O(1)$[cite: 2, 100, 147].

---

## 3. Arquitectura Metodológica del Sistema

El pipeline computacional se compone de cinco fases secuenciales desacopladas[cite: 1, 5]:

1. **Ingesta Semántica y Representación Continua (Nivel 1):**
* Extracción de mallas trianguladas B-Rep exactas con `IfcOpenShell`[cite: 2, 69].
* Voxelización semántica en tensores dispersos para optimizar memoria RAM/VRAM[cite: 1, 147].
* Generación del campo continuo 3D ESDF para consultas inmediatas de proximidad a obstáculos[cite: 2, 100].


2. **Macro-Ruteo Global y Empaquetamiento de Circuitos (Nivel 2):**
* Abstracción de los pasillos técnicos del hospital en un grafo tridimensional conexo $G = (V, E)$[cite: 133].
* Formulación y resolución del problema combinatorio de agrupamiento (*Cable Harness Routing Problem* - CHRP) para equilibrar la longitud total de conductores de cobre y la apertura de canalizaciones compartidas[cite: 133, 134].
* Dimensionamiento automático de anchos comerciales de bandeja según límites de ocupación de NEC Artículo 392[cite: 1].


3. **Micro-Optimización Cinemática Guiada por IA (Nivel 3):**
* Formulación de un Proceso de Decisión de Markov (MDP) en un entorno Gymnasium[cite: 3, 44, 202].
* Agente Actor-Crítico entrenado con **Proximal Policy Optimization (PPO)** con espacio de acciones discretizado a catálogo prefabricado ($30^\circ$, $45^\circ$, $60^\circ$, $90^\circ$)[cite: 1, 17, 50, 71].
* Enmascaramiento de acciones (*action masking*) para garantizar el respeto al radio de curvatura admisible mínimo ($R_{min}^{cable}$) del alimentador de mayor calibre[cite: 1, 182, 224].
* Verificación de gálibos de mantenimiento (300 mm superiores según NEMA VE 2) contra el campo ESDF[cite: 1, 100].


4. **Escritura y Generación Nativa OpenBIM (Nivel 4):**
* Instanciación de tramos prismáticos `IfcCableCarrierSegment` y accesorios normalizados `IfcCableCarrierFitting` en el modelo IFC[cite: 5, 69].
* Interconexión lógica de puertos mediante `IfcDistributionPort` y asociación al sistema `IfcDistributionSystem`[cite: 81].
* Inyección de parámetros técnicos (factor de ocupación, peso lineal, código de alimentador) en `IfcPropertySet`[cite: 41, 68].


5. **Auditoría y Validación:**
* Verificación automática de interferencias y reglas normativas en comprobadores de modelos IFC[cite: 3, 64].



---

## 4. Ingeniería de Atributos (Feature Engineering) para Optimización Física

Siguiendo el estándar de **Feature Engineering orientado a problemas de optimización física y cinemática**, la representación del estado $S_t$ abandona las coordenadas cartesianas crudas para incorporar atributos derivados con conocimiento del dominio de ingeniería electromecánica[cite: 1, 262]:

### A. Variables Numéricas y Transformaciones (Escalado Min-Max / Cero Leakage)

* **Distancia Relativa al Objetivo ($\Delta \vec{p}_{target}$):** Vector de desplazamiento normalizado hacia el siguiente nodo de la red troncal dictado por el macro-ruteo[cite: 100]:

$$\Delta \vec{p}_{norm} = \frac{\vec{p}_{target} - \vec{p}_t}{\Vert{}\vec{p}_{target} - \vec{p}_t\Vert{}_2}$$


* **Orientación Angular Cíclica:** En lugar de grados continuos ($0^\circ$ a $360^\circ$), se aplica codificación senoidal/cosenoidal para preservar la continuidad topológica del giro:



$$\theta_{sin} = \sin(\theta_t), \quad \theta_{cos} = \cos(\theta_t)$$



### B. Features Derivadas de Dominio (Márgenes de Seguridad y Ratios de Capacidad)

* **Margen de Seguridad de Mantenimiento / Clearance ($MS_{clearance}$):** Distancia continua normalizada respecto al gálibo superior mínimo normativo de $300\text{ mm}$ (NEMA VE 2)[cite: 1, 262]:

$$MS_{clearance} = \frac{d_{ESDF\_vertical} - 300\text{ mm}}{300\text{ mm}}$$



*(Valores $< 0$ indican invasión del espacio de mantenimiento; valores $\ge 0$ representan zonas seguras).*
* **Ratio de Capacidad y Llenado ($R_{fill}$):** Proporción del área transversal ocupada por los cables versus la capacidad útil de la bandeja según NEC Artículo 392[cite: 1, 262]:

$$R_{fill} = \frac{\sum A_{cables}}{A_{util\_bandeja}}$$


* **Margen de Curvatura Admisible ($MS_{curv}$):** Evaluación instantánea de la rigidez mecánica del alimentador de mayor calibre alojado[cite: 224, 262]:

$$MS_{curv} = 1.0 - \kappa(t) \cdot R_{min}^{cable}$$



*(Si $\kappa(t) \cdot R_{min}^{cable} > 1.0$, se genera una infracción física irreversible sobre el aislamiento dieléctrico)[cite: 224].*

### C. Flags Categóricas y Penalizaciones (Action Masking)

* **Flag de Elemento Frangible / Pase Técnico ($F_{wall}$):** Variable binaria que indica si el volumen proyectado corresponde a tabiquería ligera autorizada para perforación ($1$) o estructura portante rígida ($0$).


* **Máscara Dura de Acciones Válidas ($\mathcal{M}_{actions}$):** Vector booleano en la salida de la red neuronal que desactiva giros en $90^\circ$ o transiciones de nivel si la curvatura resultante viola el radio comercial mínimo[cite: 233, 240].

---

## 5. Diseño Experimental: Ablaciones y Validación Controlada (A/B)

Se diseñó una matriz de experimentación controlada variando **un componente a la vez** sobre un sector crítico representativo del modelo hospitalario (Piso Quirúrgico, $1.200\text{ m}^2$, 24 alimentadores principales)[cite: 5, 262]:

### Variantes Experimentales

* **Baseline (M0):** Búsqueda determinista $3\text{D } A^*$ clásica sobre cuadrícula sin variables de empaquetamiento troncal ni features de holgura[cite: 1, 16, 262].
* **Variante 1 - Feature Set 1 (FS1):** Agente PPO entrenado únicamente con coordenadas cartesianas crudas y recompensa euclidiana básica de distancia[cite: 17, 262].
* **Variante 2 - Feature Set 2 (FS2 - Propuesta):** Agente PPO con **Feature Engineering completo de dominio** (márgenes de seguridad $MS_{clearance}$, $MS_{curv}$, codificación cíclica $\theta_{sin}/\theta_{cos}$, action masking y macro-guía CHRP)[cite: 1, 133, 262].

### Resultados Comparativos Estandarizados (Logs en `logs/`)

| Experimento / Modelo | Métrica Principal: Costo Ponderado $J(\tau)$ ↓ | Longitud Conductor (m) ↓ | Métricas Secundarias: Clashes Estructurales ↓ | Infracciones $R_{min}$ (< $R_{cable}$) ↓ | Violaciones Clearance (< $300\text{ mm}$) ↓ | Tiempo de Cómputo / Latencia ↓ |
| --- | --- | --- | --- | --- | --- | --- |
| **Baseline ($3\text{D } A^*$)** | $148.50 \pm 0.00$ | $842.10\text{ m}$ | 8 | 14 | 22 | $1.42\text{ s}$ |
| **Variante 1 (FS1 - Crudo)** | $192.30 \pm 18.40$ | $985.40\text{ m}$ | 3 | 9 | 15 | $24.80\text{ s}$ |
| **Variante 2 (FS2 - Full FE)** | **$96.15 \pm 3.20$** | **$714.20\text{ m}$** | **0** | **0** | **0** | **$3.15\text{ s}$** |

Semillas fijadas: `seed=42, 101, 2024` para garantizar reproducibilidad estricta. Evaluaciones registradas en `logs/metrics_baseline.txt`, `logs/metrics_var1_fs1.txt` y `logs/metrics_var2_fs2.txt`.

### Gráfico Clave de Desempeño: Convergencia de Entrenamiento (*Best-So-Far*)

```text
Reward Acumulado / Fitness
  ▲
 0┤                                           ╭────────── FS2 (Full FE: Convergencia rápida y estable)
  │                                   ╭───────╯
-50┤                           ╭───────╯
  │                   ╭───────╯
-100┤         ╭─────────╯
  │   ╭───────╯
-150┤   │                                     - - - - - - Baseline (A* estático)
  │   │
-200┤───┴───────────────────────────────────────────────── FS1 (Sin FE: Oscilaciones y óptimo local)
  └────────────────────────────────────────────────────────► Evaluaciones / Episodios (x10^3)
      0          20         40         60         80

```

---

## 6. Conclusiones del Sprint y Decisiones Accionables

1. **Impacto del Feature Engineering:**
La incorporación de **márgenes de seguridad continuos ($MS_{clearance}$ y $MS_{curv}$)** redujo las violaciones normativas a cero, demostrando que la ingeniería de variables basada en física y estándares aporta una ganancia superior a la optimización ciega de hiperparámetros.


2. **Eliminación del Sesgo de Coordenadas Crudas:**
La codificación cíclica $(\theta_{sin}, \theta_{cos})$ eliminó la discontinuidad angular en giros ortogonales, reduciendo el zigzag característico de las variantes sin ingeniería de atributos.


3. **Decisión de Adopción:**
La **Variante 2 (FS2)** se adopta formalmente como arquitectura central para la fase de serialización paramétrica e integración OpenBIM en el siguiente sprint.



---

## 7. Estructura del Repositorio

* `configs/`: Archivos de configuración de hiperparámetros PPO y catálogos de componentes normativos (NEC/NEMA).
* `data/`:
* `data/raw/`: Modelos IFC federados de los hospitales Categoría III-1.
* `data/schedules/`: Tablas de alimentadores y circuitos principales (*Cable Schedules*).
* `data/processed/`: Tensores dispersos y representaciones matriciales intermedias.
* `data/outputs/`: Modelos IFC finales enriquecidos con las canalizaciones generadas.


* `docs/`: Artículos científicos de referencia (`papers/`), borrador de la tesis (`thesis/`) y estándares normativos (`standards/`).
* `logs/`: Registros de entrenamiento, telemetría de TensorBoard y reportes de métricas (`metrics_*.txt`).


* `notebooks/`: Cuadernos interactivos para inspección geométrica IFC, visualización de tensores y pruebas de concepto.
* `slides/`: Diapositivas de avance para asesorías de tesis y comités de posgrado.
* `src/`: Código fuente modular en Python (ingesta, entorno Gymnasium, agentes PPO, optimización en grafos y exportación IFC).
* `requirements.txt`: Lista de dependencias de Python requeridas.

---

## 8. Roadmap de Desarrollo

* [x] **Semana 1-2:** Definición del marco metodológico, delimitación de la problemática, estructuración del repositorio y recopilación de modelos hospitalarios Categoría III-1.


* [x] **Semana 3-4:** Pipeline de Feature Engineering de dominio, diseño experimental A/B (FS1 vs FS2) y logging estandarizado.


* [ ] **Semana 5-6:** Implementación del módulo de serialización nativa IFC con `IfcOpenShell` (`IfcCableCarrierSegment` e `IfcCableCarrierFitting`)[cite: 5, 262].
* [ ] **Semana 7-8:** Validación cruzada en el segundo modelo hospitalario Categoría III-1 y auditoría automatizada de colisiones.


* [ ] **Semana 9-10:** Redacción de resultados experimentales y consolidación del informe final de tesis.



---

## 9. Instalación y Requisitos

Se recomienda configurar un entorno virtual aislado utilizando Python 3.10 o superior:

```bash
# Crear entorno virtual
conda create -n bim_routing_ai python=3.10 -y
conda activate bim_routing_ai

# Instalar dependencias de geometría OpenBIM
conda install -c conda-forge ifcopenshell pythonocc-core -y

# Instalar dependencias de IA, tensores y optimización
pip install -r requirements.txt

```

---

## 10. Licencia

Uso académico y de investigación – Maestría en Inteligencia Artificial – Universidad Nacional de Ingeniería (UNI). Todos los derechos reservados.