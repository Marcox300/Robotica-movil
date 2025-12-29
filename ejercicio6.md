# Marker Based Visual Loc P6

# Objetivo

El objetivo de esta práctica es implementar un sistema de localización visual basado en AprilTags para estimar la posición y orientación de un robot en un entorno conocido, 
y combinar esta información con odometría para mantener la pose estimada incluso cuando no se detectan marcas visuales. Además, el robot se debe desplazar de forma aleatoria.

## Localización

1. Detección de AprilTags:
  Usamos el código de ejemplo para detectar los tag. Tras esto analizamos su tamaño y escogemos al más grande.

2. Cálculo de la cámara:
- Estado 0 –
  Se define la matriz de cámara usando la distancia focal (`focal_length`) y el centro de la imagen.  
  `focal_length` se calcula como una proporción del ancho de la imagen (`size[1] * 0.87`) y representa la distancia focal de la cámara en píxeles,
  necesaria para proyectar correctamente los puntos 3D del tag a coordenadas 2D de imagen.

3. Resolución del PnP (Perspective-n-Point):
  Se define la posición 3D de los vértices del tag en su propio sistema de coordenadas (`obj_points`),
  usamos `TAG_SIZE = 0.24` que representan el tamaño real del Tag en metros como menciona el enunciado.
  Tras esto, se resuelven las matrices de rotación y traslación tag -> cámara usando `cv2.solvePnP`.

4. Cálculo de la pose absoluta del robot usando visión:
  Una vez obtenido el vector de traslación y la matriz de rotación del tag respecto a la cámara (`t_tc`, `R_tc`), se transforma esta información al sistema del robot.  
  `t_robot` representa la posición del robot relativa a la cámara (suponiendo que la cámara está montada sobre el robot), adaptando los ejes:
     - X_robot = -Z_cámara  
     - Y_robot = -X_cámara  
     - Z_robot = 0 (ignoramos la altura)

   > **Nota:** estos fueron calculados a mano por simplicidad en el ejercicio.

   Tras esto calculamos matriz de rotación robot->cámara.
   Con estas matrices calculamos Matriz rotacion robot->mapa = tag->mapa * robot->camara * camara->tag. Tras esto calculamos matriz rotación traslación robot->map añadiendo la traslación.
   Finalmente, la pose absoluta del robot en el mapa se representa con la matriz de rotación y traslación 4x4 `RT_robot_map`, de la cual se extraen la posición `(x_est, y_est)` y orientación `yaw_est`.
   > **Nota:** `yaw_est` es calculada directamente de  Matriz rotacion robot->mapa por simplicidad.

5. Actualización de la pose usando odometría si no hay visión:
  Cuando no se detectan tags, se estima el movimiento del robot usando odometría (`odom` y `odom_last`).  
  Se calcula el desplazamiento incremental:  
     - `dx = odom.x - odom_last.x`  
     - `dy = odom.y - odom_last.y`  
     - `dyaw = odom.yaw - odom_last.yaw`
  
   Finalmente sumamos el incremento a la posición anterior estimada.

Como posibles mejoras:
- Filtrado de mediciones erráticas: se debería añadir un filtro que desestime calculos muy diferentes en visión.

- Fusión de sensores: en la propia visión se podría añadir una mezcla entre odometria y visión.
  Esto no se ha aplicado por la precisión actual que obtenemos en visión, solo usamos odom ante cadencias de marcas.

- Reconocimiento de múltiples tags: en lugar de usar solo el tag más grande, promediar posiciones de varios tags detectados para mejorar la precisión.

## Movimiento

El comportamiento del robot se compone en estos tres estados:

- Estado 0 --- Avanzando:
El robot se desplaza hacia adelante de manera constante. Si detecta un obstáculo frente a él (con el sensor láser), detiene el avance y cambia al estado de cálculo de giro.

- Estado 1 --- Cálculo de giro:
Se selecciona un ángulo de giro aleatorio dentro de un rango máximo permitido. A continuación, se pasa al estado de giro.

- Estado 2 --- Girando:
El robot rota hacia la orientación objetivo calculada. El giro es proporcional a la diferencia entre la orientación actual y la deseada, de modo que se ajuste suavemente. Cuando alcanza el ángulo objetivo dentro de un umbral de tolerancia, vuelve al estado de avance.

# Video
