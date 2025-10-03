# Ejercicio 1 Rumba

## Planteamiento
El objetivo de este ejercicio es crear una aspiradora de estados finitos reactivos, de tal manera que cuente con al menos 3 estados (avanzando, retrocediendo, girando).
Para implementar esta parte he creado una secuencia de estados que cambia según ciertas condiciones temporales según cada tick.
Por Tanto la lógica es avanzar de forma indefinida hasta chocar, momento que retrocede un número de Ticks determinado y gira aleatoriamente entre [-180,180] grados (aproximadamente)
utilizando un número aleatorio para definir la dirección (+-) y otro para el número de ticks. Tras esto continua avanzando.

## Extra
Mi objetivo es implementar la acción de espiral, de tal forma que barra más eficientemente zonas amplias.

### Desarrollo
En primer lugar añadí un estado para realizar la espiral limitando el número de ticks del movimiento de avanzar y haciendo general la revisión de chocar.
Tras esto realice varios planteamientos de como realizar la espiral y teniendo en cuenta que el giro tendría que ser una mezcla entre velocidades en *V* y *W*
estos fueron las ideas que tuve y sus cadencias:

- Modificar la velocidad *W* y posteriormente *V*, el error está en el poco control y razonamiento. Con el tiempo podría invertirse la espiral o alargarse
demasiado dependiendo de si aumenta o disminuye por cada tick.

- Alternar entre giros y rectas de forma proporcional. Difícil de implementar de forma sencilla al requerir más precisión proporciones constantes equivalen a mantener un giro uniforme.

### Solución
Finalmente me replantee el problema, analice la curva y es que *R* (radio) *= V/W* por tanto y como no debo aumentar en exceso *V* lo fije a 0.5 velocidad que se ajusta al mundo real.
Por su parte necesito una W menor a 1 para mantener la velocidad razonable. Por otro lado necesito que el radio aumenta con el tiempo de tal forma que obtuve dos posibilidades:

1. Por cada vuelta girar 90º desplazarse un poco hacia delante y deshacer el giro. No quise implementar el guiado del giro al no contar con precisión según se especifica en el ejercicio.
2. Aumentar el radio por cada tick.

Esta última opción fue la que escogí.
Simplemente razonando que modificamos *W* y ajustamos los valores iniciales para que la velocidad no sea mayor a la estipulada de 1. Por último simplemente adaptamos esta fórmula al programa y obtenemos la solución:

*R = V/W*   *R0 = V/W0*   *W = V / R_NEW*   *R_NEW = R_LAST + K***T*

siendo K una proporción a ajustar y T el contador de Ticks como en mi caso es 1 la T se puede omitir. 
