# Segmentación de la demanda de taxis en Nueva York

Tres marcos de aprendizaje no supervisado —teoría de la información, clustering por partición y mezclas gaussianas— aplicados a 1.46 millones de viajes para responder una pregunta operativa: **dónde, cuándo y por qué se concentra la demanda**.

Proyecto 2 · Aprendizaje de Máquina No Supervisado · Universidad de La Sabana

| Rol | Integrante |
|---|---|
| Rol 1 — Incertidumbre e información | Santiago Parada |
| Rol 2 — Segmentación por partición | Francisco Castro |
| Rol 3 — Mezclas gaussianas y variables latentes | Natalia Rodriguez |
| Rol 4 — Negocio, datos e integración | David Cifuentes |

---

## El problema de negocio

Una flota de taxis necesita decidir dónde reposicionar vehículos, cómo dimensionar turnos y cómo estimar tiempos de viaje por zona y franja horaria. Este proyecto integra tres lentes no supervisadas sobre el mismo dataset para convertir esas decisiones en reglas basadas en datos, en vez de intuición operativa.

## El dataset

**New York City Taxi Trip Duration** — competencia de Kaggle basada en los registros de 2016 del NYC Yellow Cab, publicados originalmente por la NYC Taxi and Limousine Commission (TLC) vía Google BigQuery.
https://www.kaggle.com/c/nyc-taxi-trip-duration

- **1,458,644** viajes originales · 11 columnas (coordenadas de recogida/destino, timestamps, pasajeros, proveedor)
- Limpieza con reglas explícitas (`data/registro_limpieza.csv`): coordenadas fuera de NYC, duración fuera de 60 s–3 h, pasajeros fuera de 1–6, distancia < 0.1 km, velocidad > 100 km/h → **1,439,504 viajes restantes** (1.31 % eliminado)
- Zonificación geográfica en **20 zonas** (`data/centroides_zonas.csv`) mediante clustering de coordenadas
- Muestra de **50,000 viajes** (`data/muestra_50k.csv`) usada por los cuatro roles: el Rol 1 calcula todas sus entropías sobre ella, y es la base común de los Roles 2, 3 y 4, por el costo computacional de DBSCAN y EM sobre el dataset completo

> **`train.csv` (192 MB) y `test.csv` (68 MB) no están incluidos en este repositorio** por exceder el límite de 100 MB de GitHub. Para reproducir el pipeline desde cero, descárgalos de Kaggle y colócalos en `data/`.

## Metodología: tres marcos, un problema

| Notebook | Rol | Herramientas |
|---|---|---|
| [`00_datos.ipynb`](notebooks/00_datos.ipynb) | Preparación | Limpieza, variables derivadas, zonificación, muestreo |
| [`01_rol1_informacion.ipynb`](notebooks/01_rol1_informacion.ipynb) | Incertidumbre e información | Entropía, entropía condicional, información mutua, KL, entropía cruzada, desigualdad de Jensen, log-sum-exp |
| [`02_rol2_segmentacion.ipynb`](notebooks/02_rol2_segmentacion.ipynb) | Segmentación por partición | K-means, clustering jerárquico (Ward), DBSCAN |
| [`03_rol3_mezclas.ipynb`](notebooks/03_rol3_mezclas.ipynb) | Mezclas gaussianas | Algoritmo EM implementado desde cero, validado contra scikit-learn |
| [`04_rol4_negocio.ipynb`](notebooks/04_rol4_negocio.ipynb) | Integración | Combina los tres bloques en cuatro recomendaciones operativas |

---

## Hallazgos principales

### 1. El origen predice el destino; la hora casi no aporta

La información mutua entre zona de origen y destino es **0.377 bits**, mientras que entre hora y destino es apenas **0.0717 bits, once veces el nivel de azar estimado por permutación (0.0063 bits, p < 0.002), pero pequeño en magnitud frente a los 0.377 bits del origen**. Saber de dónde sale un viaje reduce mucho más la incertidumbre sobre a dónde va que saber a qué hora ocurre.

### 2. Fin de semana y entre semana difieren en el cuándo, no en el dónde

La divergencia KL entre la distribución de fin de semana y la de días laborales es **0.188 bits** por hora, pero solo **0.050 bits** por zona. El patrón geográfico de la demanda se mantiene; lo que cambia es la curva horaria.

### 3. K-means encuentra 4 segmentos operativos; DBSCAN los rechaza como densidad

| Método | Resultado |
|---|---|
| K-means | K = 4, elegido por codo de inercia y mínimo local de Davies-Bouldin (la silueta prefería K = 2, pero con valor bajo ~0.27 y sin separar más que viajes cortos de largos) |
| Ward (jerárquico) | ARI = 0.358 frente a K-means, sobre 5,000 viajes |
| DBSCAN | eps = 0.439 (codo de k-distancia), solo 2 clusters con 1.69 % de ruido |

DBSCAN no reproduce los 4 segmentos: el espacio de características no tiene la estructura de densidad que ese algoritmo necesita, evidencia de que las diferencias entre grupos son graduales, no huecos de densidad.

### 4. Las mezclas gaussianas confirman la partición y aportan incertidumbre

El GMM selecciona **4 perfiles** (BIC monótono, máxima estabilidad entre mitades de la muestra), con EM implementado desde cero convergiendo en 46 iteraciones y validado contra scikit-learn (diferencia relativa **1.3×10⁻⁷**). La inicialización con k-means++ converge un 39 % más rápido que la aleatoria (66.0 vs. 108.2 iteraciones promedio). El acuerdo con los clusters duros de K-means es moderado (ARI = 0.375) y solo el 1.8 % de los viajes queda con responsabilidad ambigua (< 0.6), lo que indica fronteras razonablemente nítidas entre perfiles.

## Perfiles operativos (K-means, Rol 2)

| Cluster | % viajes | Distancia mediana | Duración mediana | Franja dominante |
|---|---|---|---|---|
| 0 | 18.0 % | 8.3 km | 23.7 min | Noche |
| 1 | 26.9 % | 1.5 km | 8.3 min | Valle |
| 2 | 26.2 % | 2.3 km | 14.7 min | Pico PM |
| 3 | 28.8 % | 1.6 km | 7.2 min | Noche |

## Recomendaciones de negocio (Rol 4)

1. **Reposicionamiento nocturno**: entre la 1:00 y las 3:00 am la demanda deja de repartirse uniformemente y se concentra en las zonas 6, 7 y 9 —de viajes cortos nocturnos—, que cubren el 42.6 % de esa ventana frente al 29.4 % de las tres mejores zonas de día.
2. **Calendario operativo por patrón**, ajustando turnos a la curva horaria de cada perfil en vez de a promedios agregados.
3. **Estimación de tiempos por franja y zona**, usando los perfiles de duración/velocidad como base en vez de un tiempo promedio único.
4. **Tratamiento diferenciado de viajes atípicos**, separando el ruido operativo de la señal real de demanda.

Detalle completo y evidencia numérica en [`04_rol4_negocio.ipynb`](notebooks/04_rol4_negocio.ipynb).

---

## Limitaciones

- Los cuatro roles trabajan sobre una muestra de 50,000 viajes, no el dataset completo, por el costo computacional de DBSCAN y EM.
- Ward se validó solo sobre 5,000 viajes por la complejidad O(n²) del clustering jerárquico aglomerativo.
- El acuerdo moderado entre los tres métodos —ARI de 0.358 entre Ward y K-means, y de 0.375 entre GMM y K-means— refleja que son razonables pero no hay una partición única "correcta": los segmentos son continuos, no discretos por naturaleza.

## Estructura del repositorio

```
├── data/
│   ├── train.csv                      # NO incluido (192 MB) — descargar de Kaggle
│   ├── test.csv                       # NO incluido (68 MB) — descargar de Kaggle
│   ├── muestra_50k.csv                # muestra común usada por los cuatro roles
│   ├── sample_submission.csv
│   ├── registro_limpieza.csv          # reglas de limpieza y registros eliminados
│   ├── centroides_zonas.csv           # centroides de las 20 zonas geográficas
│   ├── resultados_rol1.csv            # tabla de resultados — Rol 1
│   ├── resultados_rol2.csv            # tabla de resultados — Rol 2
│   ├── resultados_rol3.csv            # tabla de resultados — Rol 3
│   ├── resultados_consolidados.csv    # resultados de los tres roles
│   ├── metricas_rol2.csv              # métricas de validación de clustering
│   ├── etiquetas_rol2.csv             # cluster asignado por viaje
│   ├── perfiles_rol2.csv              # perfiles operativos K-means
│   ├── perfiles_rol3.csv              # perfiles latentes GMM
│   └── perfiles_viajes_rol3.csv       # responsabilidades por viaje
├── notebooks/
│   ├── 00_datos.ipynb
│   ├── 01_rol1_informacion.ipynb
│   ├── 02_rol2_segmentacion.ipynb
│   ├── 03_rol3_mezclas.ipynb
│   └── 04_rol4_negocio.ipynb
└── README.md
```

## Cómo reproducir

```bash
# 1. Descargar train.csv y test.csv de Kaggle y colocarlos en data/
#    https://www.kaggle.com/c/nyc-taxi-trip-duration

# 2. Instalar dependencias
pip install pandas numpy scipy scikit-learn matplotlib seaborn jupyter

# 3. Ejecutar los notebooks en orden
jupyter notebook notebooks/00_datos.ipynb
```

Los notebooks 01–04 leen los archivos ya procesados en `data/`, por lo que pueden ejecutarse de forma independiente sin repetir la limpieza — solo `00_datos.ipynb` requiere `train.csv` completo.

**Stack:** Python · pandas · NumPy · SciPy · scikit-learn · matplotlib

## Referencia

New York City Taxi and Limousine Commission (TLC). (2017). *New York City Taxi Trip Duration* [Data set]. Kaggle. https://www.kaggle.com/c/nyc-taxi-trip-duration
