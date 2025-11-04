# Ejercicio 3---VFF

# Planteamiento
Programa un coche de Fórmula1, equipado con un sensor laser y un sensor GPS, para que recorra un circuito de carreras con valla en los laterales y con otros coches como obstáculos a lo largo del recorrido.

# Desarrollo

Para abordar este ejercicio se desarrollaron varios módulos que trabajan de forma conjunta. El flujo general del programa se basa en:

1. Leer la posición actual del robot y la del objetivo.
2. Convertir los datos del láser en coordenadas cartesianas.
3. Calcular un vector repulsivo en base a los obstáculos.
4. Calcular un vector atractivo hacia el objetivo.
5. Fusionar ambos vectores para obtener el vector resultante de movimiento.
6. Aplicar control sobre las componentes X e Y para generar velocidades V y W.

## El módulo **vectorial** se encarga de generar fuerzas proporcionales que guían el movimiento del robot.

## La fuerza atractiva
Se calcula aplicando la matriz de transformación sobre la posición objetivo y obteniendo un módulo que determina la distancia máxima hacia el punto deseado.

## La fuerza repulsiva 
Transforma las mediciones del LiDAR en ángulos y, posteriormente, invierte sus valores para representar la dirección opuesta al obstáculo.
En este proceso se toma como referencia la distancia medida y se aplica la expresión exponencial
$$
mag = A \cdot e^{-k \cdot \frac{dist}{fac}}
$$
- `mag`: magnitud de la fuerza repulsiva resultante.  
- `A`: factor de escala que ajusta la intensidad máxima de la fuerza.  
- `k`: constante de decaimiento exponencial que controla la velocidad con la que disminuye la fuerza con la distancia.  
- `dist`: distancia entre el robot y el obstáculo detectado.  
- `fac`: factor angular dependiente del ángulo del rayo LiDAR, que aumenta la influencia de los obstáculos situados frente al robot.

Con esta última variable (`fac`) podemos regular la influencia de los muros y obstaculos laterales.
![Efecto del parámetro fac](img/vff.png)
En la imagen se puede ver que el área de influencia de la fuerza repulsiva se ha definido según la distancia entre el centro de la carretera y los muros.
También se muestra el rango aproximado en el que actúa la repulsión.
Al usar una función exponencial, el coche puede acercarse más a los muros sin que la fuerza crezca demasiado rápido.
Gracias a esto, puede seguir una trazada más natural y mantener mayor velocidad en zonas donde no hay peligro de colisión.

## La fusión de fuerzas
El objetivo es generar un vector que define la dirección y la velocidad del movimiento.


Video:
<video width="640" height="360" controls>
  <source src="video/vff.mp4" type="video/mp4">
  Tu navegador no soporta el video.
</video>
