# Robot de Mapeo Probabilístico en P5

Este proyecto implementa un robot autónomo equipado con sensor láser y autolocalización para explorar y mapear una nave industrial, generando un mapa de ocupación probabilístico en forma de rejilla.

# Objetivo

El robot debe:

- Explorar de manera autónoma todo el entorno, visitando zonas desconocidas.

- Construir un mapa probabilístico basado en las lecturas del sensor láser y la posición estimada por el sistema de autolocalización.

- Visualizar el mapa actualizado en tiempo real.

## Mapeo probabilístico

El robot mantiene un gridmap en el que cada celda representa un área del entorno y su probabilidad de estar ocupada. Para actualizar las celdas, se implementa:

- Modelo de sensor: La probabilidad de ocupación a partir de la observación, 
```𝑝(ocupada∣obs(𝑡))p(ocupada∣obs(t))```, cada lectura del láser se traduce en celdas libres o ocupadas usando Ratios de probabilidad (log-odds).
Esta combinación de evidencias con Bayes esta limitada por -1 a 254 de tal forma que <= 0 es espacio libre y >=1 es espacio ocupado, el valor acumulado de los espacios libres es menor para adaptarse a cambios del entorno más rápido.

- Mediciones condicionadas al movimiento: el robot solo actualiza el mapa si se ha desplazado o girado lo suficiente desde la última medición, evitando redundancia y reduciendo el error acumulado.

- Algoritmo de Bresenham: se utiliza para trazar la línea entre el robot y el punto detectado por el láser, marcando como libres las celdas por las que pasa el rayo y como ocupadas las celdas donde detecta un obstáculo.
A esto se le suma que se usa el rango máximo del laser para delimitar zonas libre.

Esta estrategia permite construir un mapa robusto incluso con sensores imperfectos, aunque la calidad del mapa se ve afectada si la autolocalización tiene mucho ruido. No funciona bien con Odom2 y Odom3

## Algoritmo de exploración y movimiento

La navegación del robot se organiza en dos partes principales:

1. Barrido automático:

El robot avanza mientras el frente está libre. Al encontrar un obstáculo, gira 90° y avanza un tramo corto antes de continuar, creando un patrón de exploración tipo zigzag. El tramo corto debe ser menor sin el apartado 2.
Este comportamiento permite cubrir grandes zonas de manera sistemática y evitando el ruido de forma efectiva.

2. Búsqueda de zonas desconocidas:

Cuando el robot se encuentra en una posición sin salidas libres o zonas por explorar, utiliza una estrategia de búsqueda basada en celdas seguras no exploradas.
Se calcula una ruta hacia estas zonas utilizando A* sobre el gridmap probabilístico, evitando obstáculos conocidos y manteniendo un margen de seguridad.
Una vez alcanzada la zona, se retoma el patrón de barrido.

Este enfoque combinado garantiza que el robot pueda cubrir todo el entorno de manera eficiente, evitando quedarse atrapado y maximizando la información recolectada para el mapa.

El algoritmo A* puede dar problemas en la escalabilidad, es por esto que he tenido que limitarlo. Esto se debe a que la ruta tiene que garantizar que puede pasar el robot almenos en las zonas ya mapeadas.

# Observaciones y pruebas
