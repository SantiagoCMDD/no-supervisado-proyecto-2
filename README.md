Segmentación de la demanda de taxis en Nueva York

Tres marcos de aprendizaje no supervisado —teoría de la información, clustering por partición y mezclas gaussianas— aplicados a 1.46 millones de viajes para responder una pregunta operativa: dónde, cuándo y por qué se concentra la demanda.

Proyecto 2 · Aprendizaje de Máquina No Supervisado · Universidad de La Sabana

Rol	Integrante
Rol 1 — Incertidumbre e información	Santiago Parada
Rol 2 — Segmentación por partición	Francisco Castro
Rol 3 — Mezclas gaussianas y variables latentes	Natalia Rodriguez
Rol 4 — Negocio, datos e integración	David Cifuentes
El problema de negocio

Una flota de taxis necesita decidir dónde reposicionar vehículos, cómo dimensionar turnos y cómo estimar tiempos de viaje por zona y franja horaria. Este proyecto integra tres lentes no supervisadas sobre el mismo dataset para convertir esas decisiones en reglas basadas en datos, en vez de intuición operativa.

El dataset

New York City Taxi Trip Duration — competencia de Kaggle basada en los registros de 2016 del NYC Yellow Cab, publicados originalmente por la NYC Taxi and Limousine Commission (TLC). https://www.kaggle.com/c/nyc-taxi-trip-duration

1,458,644 viajes originales · 11 columnas (coordenadas de recogida/destino, timestamps, pasajeros, proveedor)
Limpieza con reglas explícitas (data/registro_limpieza.csv): coordenadas fuera de NYC, duración fuera de 60 s–3 h, pasajeros fuera de 1–6, distancia < 0.1 km, velocidad > 100 km/h → 1,439,504 viajes restantes (1.31 % eliminado)
Zonificación en 20 zonas (data/centroides_zonas.csv) mediante K-means sobre las coordenadas de recogida proyectadas a kilómetros, comunes a los cuatro roles
Muestra de 50,000 viajes (data/muestra_50k.csv) usada por los cuatro roles, para que entropías, clusters y perfiles se refieran exactamente a los mismos viajes. Representatividad verificada con divergencia KL contra la población: menos de 0.35 milibits en hora, día y mes

train.csv (192 MB) y test.csv (68 MB) no están incluidos en este repositorio por exceder el límite de 100 MB de GitHub. Para reproducir el pipeline desde cero, descárgalos de Kaggle y colócalos en data/.

Metodología: tres marcos, un problema
Notebook	Rol	Herramientas
00_datos.ipynb	Preparación	Limpieza, variables derivadas, zonificación, muestreo
01_rol1_informacion.ipynb	Incertidumbre e información	Entropía, entropía condicional, información mutua, KL, entropía cruzada, desigualdad de Jensen, log-sum-exp
02_rol2_segmentacion.ipynb	Segmentación por partición	K-means, clustering jerárquico (Ward), DBSCAN
03_rol3_mezclas.ipynb	Mezclas gaussianas	Algoritmo EM implementado desde cero, validado contra scikit-learn
04_rol4_negocio.ipynb	Integración	Combina los tres bloques en cuatro recomendaciones operativas
Hallazgos principales
1. El origen predice el destino; la hora aporta poco

La información mutua entre zona de origen y destino es 0.377 bits, mientras que entre hora y destino es de 0.0717 bits. Ese segundo valor es real y no ruido: una prueba de 500 permutaciones sitúa el nivel de azar en 0.0063 bits (p < 0.002), así que lo observado es 11 veces mayor. Pero en magnitud queda cinco veces por debajo del aporte del origen. Saber de dónde sale un viaje reduce mucho más la incertidumbre sobre a dónde va que saber a qué hora ocurre.

2. Fin de semana y entre semana difieren en el cuándo, no en el dónde

La divergencia KL entre la distribución de fin de semana y la de días laborales es 0.188 bits por hora, pero solo 0.050 bits por zona. El patrón geográfico de la demanda se mantiene; lo que cambia es la curva horaria: la madrugada pasa del 7.0 % al 20.2 % de los viajes.

3. La demanda solo se concentra de madrugada

La entropía global de la zona de recogida es 3.992 bits, el 92.4 % del máximo posible (log₂20 = 4.322), equivalente a 15.9 zonas efectivas: de día la demanda exige cobertura amplia. Entre las 2:00 y las 3:00 cae a 3.606 bits (unas 12 zonas efectivas) y la zona 6 concentra el 23.6 % de las recogidas. Los intervalos bootstrap del 95 % confirman que esa caída no se cruza con ninguna hora entre las 6:00 y las 23:00.

4. K-means encuentra 4 segmentos operativos; DBSCAN funciona como detector de atípicos
Método	Resultado
K-means	K = 4 (silueta 0.232), elegido por codo de inercia y mínimo local de Davies-Bouldin (1.295). La silueta habría preferido K = 2 (0.269); se deja constancia
Ward (jerárquico)	ARI = 0.358 frente a K-means, sobre 5,000 viajes; coincide en el grupo de viajes largos
DBSCAN	eps = 0.439 (codo de k-distancia), min_samples = 10; 2 clusters y 1.69 % de ruido

DBSCAN no reproduce los 4 segmentos: el espacio de características no tiene la estructura de densidad que ese algoritmo necesita, evidencia de que las diferencias entre grupos son graduales y no huecos de densidad. Su utilidad aquí es identificar atípicos: los 843 viajes de ruido tienen velocidad mediana de 36.9 km/h frente a 12.8 km/h del resto y se concentran en aeropuertos (7.8 % de los viajes desde JFK).

5. Las mezclas gaussianas confirman la partición y aportan incertidumbre

El GMM selecciona 4 perfiles. El BIC decrece de forma monótona hasta K = 12, así que la selección se hizo por estabilidad: con K = 4 dos mitades independientes de la muestra coinciden con ARI = 0.966, el máximo del rango. El EM está implementado desde cero, converge en 46 iteraciones y está validado contra scikit-learn (diferencia relativa 1.3×10⁻⁷). La inicialización con k-means++ converge un 39 % más rápido que la aleatoria (66.0 vs. 108.2 iteraciones promedio), aunque solo 1 de cada 10 semillas alcanza el mejor óptimo con cualquiera de los dos métodos. El acuerdo con los clusters duros de K-means es moderado (ARI = 0.375) y solo el 1.8 % de los viajes queda con responsabilidad ambigua (< 0.6).

Perfiles operativos (K-means, Rol 2)
Cluster	% viajes	Distancia mediana	Duración mediana	Velocidad mediana	Franja dominante
0	18.0 %	8.3 km	23.7 min	21.8 km/h	Noche (origen LaGuardia)
1	26.9 %	1.5 km	8.3 min	11.6 km/h	Valle
2	26.2 %	2.3 km	14.7 min	9.6 km/h	Pico PM
3	28.8 %	1.6 km	7.2 min	13.9 km/h	Noche

Los clusters 2 y 3 recorren distancias parecidas pero el de la tarde tarda el doble: la diferencia no es el viaje, es el tráfico.

Recomendaciones de negocio (Rol 4)
Reposicionamiento nocturno. Entre 1:00 y 3:00, llevar cerca del 40 % de los conductores activos a las zonas 6, 7 y 9, que cubren el 42.6 % de la demanda de esa ventana frente al 29.4 % de las tres mejores zonas diurnas.
Calendario operativo por patrón, no por día de la semana: trasladar 13 puntos porcentuales de horas de conductor del bloque 7:00–9:00 al bloque 0:00–4:00 en días tipo fin de semana. El clasificador del Rol 1 identifica el tipo de día con 97.8 % de acierto y detecta festivos que el calendario laboral no marca.
Estimación de tiempos por franja y zona. Reemplazar la velocidad única del estimador baja el error absoluto medio de 6.78 a 5.73 minutos (−15.5 %) sobre meses no usados para estimar, con mejoras de hasta 43.8 % en madrugada.
Tratamiento diferenciado de viajes atípicos. El 1.69 % marcado por DBSCAN consume el 2.18 % de los minutos de conductor y sigue rutas predecibles (13.1 % de los viajes desde JFK terminan en Brooklyn): línea de servicio aparte y exclusión del entrenamiento de los modelos generales.

Detalle completo y evidencia numérica en 04_rol4_negocio.ipynb.

Limitaciones
El periodo cubre seis meses de 2016, así que no captura estacionalidad anual, y son viajes de taxi y no de plataforma.
Ward se validó solo sobre 5,000 viajes por la complejidad O(n²) del clustering jerárquico aglomerativo.
Los ARI moderados reflejan que no hay una partición única correcta: 0.358 entre Ward y K-means, y 0.375 entre los perfiles blandos del GMM y los clusters duros. Los segmentos son continuos, no discretos por naturaleza.
Las distancias son geodésicas (Haversine) y no de recorrido, de modo que las velocidades son efectivas en línea recta y no de velocímetro.
La zonificación de 20 zonas es una decisión operativa: las entropías en bits no son comparables con las de otra zonificación.
Estructura del repositorio
├── data/
│   ├── train.csv                      # NO incluido (192 MB) — descargar de Kaggle
│   ├── test.csv                       # NO incluido (68 MB) — descargar de Kaggle
│   ├── muestra_50k.csv                # muestra común a los cuatro roles
│   ├── registro_limpieza.csv          # reglas de limpieza y registros eliminados
│   ├── centroides_zonas.csv           # centroides de las 20 zonas geográficas
│   ├── resultados_rol1.csv            # tabla de resultados — Rol 1
│   ├── resultados_rol2.csv            # tabla de resultados — Rol 2
│   ├── resultados_rol3.csv            # tabla de resultados — Rol 3
│   ├── resultados_consolidados.csv    # resultados de los tres bloques técnicos
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
Cómo reproducir
bash
# 1. Descargar train.csv de Kaggle y colocarlo en data/
#    https://www.kaggle.com/c/nyc-taxi-trip-duration

# 2. Instalar dependencias
pip install pandas numpy scipy scikit-learn matplotlib jupyter

# 3. Ejecutar los notebooks en orden
jupyter notebook notebooks/00_datos.ipynb

Los notebooks 01–04 leen los archivos ya procesados en data/, por lo que pueden ejecutarse de forma independiente sin repetir la limpieza. Solo 00_datos.ipynb requiere train.csv completo. Todos los procesos aleatorios usan semilla fija (42), de modo que los resultados son reproducibles.

Stack: Python · pandas · NumPy · SciPy · scikit-learn · matplotlib

Referencia

New York City Taxi and Limousine Commission (TLC). (2017). New York City Taxi Trip Duration [Data set]. Kaggle. https://www.kaggle.com/c/nyc-taxi-trip-duration
