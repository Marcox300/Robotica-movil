# Marker Based Visual Loc P6

# Objetivo

El objetivo de esta práctica es implementar un sistema de localización visual basado en AprilTags para estimar la posición y orientación de un robot en un entorno conocido, 
y combinar esta información con odometría para mantener la pose estimada incluso cuando no se detectan marcas visuales. Además, el robot se debe desplazar de forma aleatoria.

## Localización

1. Detección de AprilTags:
  Usamos el código de ejemplo para detectar los tag. Tras esto analizamos su tamaño y escogemos al más grande.

2. Cálculo de la cámara:
  Se define la matriz de cámara usando la distancia focal (`focal_length`) y el centro de la imagen.  
  `focal_length` se calcula como una proporción del ancho de la imagen (`size[1] * 0.87` en este caso) y representa la distancia focal de la cámara en píxeles,
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

# Videos

Aqui se puede ver el funcionamiento:
> **Nota:** el primer video ODOM sale como INFO.

<video width="640" height="360" controls>
  <source src="video/p6_move1.webm" type="video/webm">
  Tu navegador no soporta el video.
</video>

<video width="640" height="360" controls>
  <source src="video/p6_move2.webm" type="video/webm">
  Tu navegador no soporta el video.
</video>

En este video se aprecia cómo la distancia afecta a la precisión, se aprecia que a partir del 0:54 la estela roja que deja empieza a disminuir el ruido al acercarse al tag.
Pero antes de perder al tag en el 1:10 se aprecia un salto causado por el ángulo de la cámara con el tag.

<video width="640" height="360" controls>
  <source src="video/p6_ruido.webm" type="video/webm">
  Tu navegador no soporta el video.
</video>

---
```python
import WebGUI
import HAL
import Frequency
import pyapriltags
import cv2
import numpy as np
import math
import yaml
from pathlib import Path
import random
# Enter sequential code!

MIN_RANGE_LASER = 0.2
TAG_SIZE = 0.24

MAX_TURN_RAD = math.radians(170)
MAX_W = 0.5
STOP_ANGLE_RAD = math.radians(10)

# tags
conf = yaml.safe_load(
    Path("/resources/exercises/marker_visual_loc/apriltags_poses.yaml").read_text()
)
tags = conf["tags"]

obj_points = np.array([
    [-TAG_SIZE/2, -TAG_SIZE/2, 0],
    [ TAG_SIZE/2, -TAG_SIZE/2, 0],
    [ TAG_SIZE/2,  TAG_SIZE/2, 0],
    [-TAG_SIZE/2,  TAG_SIZE/2, 0]
], dtype=np.float32)
detector = pyapriltags.Detector(searchpath=["apriltags"], families="tag36h11")

# pose
odom_last = None
pos_est = None

# states
# 0 = forward
# 1 = calculate turn
# 2 = turn
state = 0
turn_direction = 1
target_yaw = 0
print("[STATE 0] Avanzando")

# ----------------------- move -----------------------

def front_obstacle(laser_data, threshold=1):
    front_angles = list(range(350, 360)) + list(range(0, 11))

    for i in front_angles:
        dist = laser_data.values[i]
        if MIN_RANGE_LASER < dist < threshold:
            return True
    return False

def angle_diff(a, b):
    diff = (a - b + math.pi) % (2 * math.pi) - math.pi
    return diff

def move_car(vel, turn):
    HAL.setV(vel)
    HAL.setW(turn)

# ----------------------- main -----------------------
while True:
    odom = HAL.getOdom()
    laser_data = HAL.getLaserData() 

    #print("[INFO] loading image...")
    image = HAL.getImage()
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    size = image.shape
    focal_length = size[1] * 0.87
    # print(focal_length)
    center = (size[1] / 2, size[0] / 2)
    camera_matrix = np.array([
        [focal_length, 0, center[0]],
        [0, focal_length, center[1]],
        [0, 0, 1]
    ], dtype="double")
    dist_coeffs = np.zeros((4,1))

    # print("[INFO] detecting AprilTags...")
    results = detector.detect(gray)
    # print("[INFO] {} total AprilTags detected".format(len(results)))
    tag_info = []

    # loop over the AprilTag detection results
    for r in results:
        # extract the bounding box (x, y)-coordinates for the AprilTag
        # and convert each of the (x, y)-coordinate pairs to integers
        (ptA, ptB, ptC, ptD) = r.corners
        ptB = (int(ptB[0]), int(ptB[1]))
        ptC = (int(ptC[0]), int(ptC[1]))
        ptD = (int(ptD[0]), int(ptD[1]))
        ptA = (int(ptA[0]), int(ptA[1]))
        # draw the bounding box of the AprilTag detection
        cv2.line(image, ptA, ptB, (0, 255, 0), 2)
        cv2.line(image, ptB, ptC, (0, 255, 0), 2)
        cv2.line(image, ptC, ptD, (0, 255, 0), 2)
        cv2.line(image, ptD, ptA, (0, 255, 0), 2)
        # draw the center (x, y)-coordinates of the AprilTag
        (cX, cY) = (int(r.center[0]), int(r.center[1]))
        cv2.circle(image, (cX, cY), 5, (0, 0, 255), -1)
        # draw the tag family on the image
        tagFamily = r.tag_family.decode("utf-8")
        cv2.putText(
            image,
            tagFamily,
            (ptA[0], ptA[1] - 15),
            cv2.FONT_HERSHEY_SIMPLEX,
            0.5,
            (0, 255, 0),
            2,
        )
        xs = [p[0] for p in r.corners]
        ys = [p[1] for p in r.corners]
        area = (max(xs) - min(xs)) * (max(ys) - min(ys))
        tag_info.append((r.tag_id, area, r))
        # print("[INFO] tag family: {}".format(tagFamily))
        # print(f"[INFO] tag ID {r.tag_id}")

    WebGUI.showImage(image)
    if len(tag_info) > 0:

        # obtenemos el tag más grande
        tag_id, _, r = max(tag_info, key=lambda x: x[1])
        tag_name = f"tag_{tag_id}"

        if tag_name in tags:
            x_real, y_real, z_real, yaw_real = tags[tag_name]["position"]
            # print(f"[TAG REAL] X: {x_real:.2f}, Y: {y_real:.2f}, Z: {z_real:.2f}, Yaw: {yaw_real:.2f}")
            img_points = np.array(r.corners, dtype=np.float32)

            ok, rvec, tvec = cv2.solvePnP(
                obj_points,
                img_points,
                camera_matrix,
                dist_coeffs
            )

            if ok:
                # Rotation matrix and translation vector tag -> camera
                R_ct, _ = cv2.Rodrigues(rvec)
                t_ct = tvec

                # Convert from tag -> camera to camera -> tag
                R_tc = R_ct.T
                t_tc = -R_tc @ t_ct

                # Rotation-translation matrix of the tag in the map
                x_t, y_t, z_t, yaw_t = tags[tag_name]["position"]
                c = math.cos(yaw_t)
                s = math.sin(yaw_t)
                RT_tag_map = np.array([
                    [ c, -s, 0, x_t],
                    [ s,  c, 0, y_t],
                    [ 0,  0, 1, z_t],
                    [ 0,  0, 0, 1  ]
                ])

                # Translation vector camera -> robot
                # We assume the camera is the robot
                t_robot = np.zeros(3)
                t_robot[0] = -t_tc[2]    # X robot = -Z camera
                t_robot[1] = -t_tc[0]    # Y robot = -X camera
                t_robot[2] = 0            # Z robot = 0, ignore height

                # Rotation matrix robot -> camera
                R_robot_cam = np.array([
                    [1, 0, 0],  # X_robot
                    [0, 0, -1],   # Y_robot
                    [0, 1, 0]   # Z_robot
                ])

                # Rotation matrix robot -> map =
                # tag->map (rotation only) * robot->tag
                # robot->tag = robot->camera * camera->tag
                # robot -> camera -> tag -> map
                R_robot_map = RT_tag_map[:3, :3] @ R_robot_cam @ R_tc
                # print(R_robot_map)

                # Rotation-translation matrix robot -> map
                RT_robot_map = np.eye(4)
                RT_robot_map[:3, :3] = R_robot_map
                RT_robot_map[:3, 3] = RT_tag_map[:3, 3] + RT_tag_map[:3, :3] @ t_robot

                # Final position and orientation
                x_est = RT_robot_map[0, 3]
                y_est = RT_robot_map[1, 3]
                # For orientation we directly use the rotation matrix
                yaw_est = math.atan2(R_robot_map[1, 0], R_robot_map[0, 0])

                pos_est = (x_est, y_est, yaw_est)

                WebGUI.showEstimatedPose(pos_est)
                print(f"[VISION] X: {x_est:.2f}, Y: {y_est:.2f}, Yaw: {yaw_est:.2f}")

    else:
        # No vision available, use odometry
        if pos_est is not None and odom_last is not None:
            dx = odom.x - odom_last.x
            dy = odom.y - odom_last.y
            dyaw = odom.yaw - odom_last.yaw

            x_est, y_est, yaw_est = pos_est
            x_est += dx
            y_est += dy
            yaw_est += dyaw

            pos_est = (x_est, y_est, yaw_est)

            WebGUI.showEstimatedPose(pos_est)
            print(f"[ODOM] Posición estimada = X: {pos_est[0]:.2f}, Y: {pos_est[1]:.2f}, Yaw: {pos_est[2]:.2f}")

    odom_last = odom


    match state:
        case 0:
            if front_obstacle(laser_data, threshold=1):
                move_car(0, 0)
                state = 1
                print("[STATE 1] Obstáculo detectado")
            else:
                move_car(0.5, 0)

        case 1:
            delta_yaw = random.uniform(-MAX_TURN_RAD, MAX_TURN_RAD)
            target_yaw = (odom.yaw + delta_yaw) % (2 * math.pi)
            state = 2
            print(f"[STATE 2] Giro aleatorio: {math.degrees(delta_yaw):.1f}°")

        case 2:
            diff = angle_diff(target_yaw, odom.yaw)

            if abs(diff) < STOP_ANGLE_RAD:
                move_car(0, 0)
                state = 0
                print("[STATE 0] Avanzando")
            else:
                w = max(-MAX_W, min(MAX_W, 1.0 * diff))
                move_car(0, w)
                print(f"[STATE 2] Girando proporcional ({math.degrees(diff):.1f}°), w={w:.2f}")


    Frequency.tick()
```
