# Blog-Maquina-Expendedora

## Arquitectura
En esta practica tendremos 3 estados distintos, Arranque, Servicio y Administracion. En cada uno de ellos debemos analizar las posibles tareas que debemos realizar.

## Arranque (start)

El arranque es muy sencillo, tendremos que realizar lo siguiente, hacer que parpadee un led rojo encendiendolo 1 segundo y apagandolo 1 segundo, esto se repite 3 veces y se apaga, y a la vez que el led parpadea la pantalla lcd debe imprimir "Cargando ...". En este estado, podemos usar una simple tarea que cuando haga el primer parpadeo, imprima en la pantalla lcd el cargando pero luego en las siguientes iteraciones no haga nada.

El thread del led debera manejar una variable contador, que aumentara cada vez que encienda o se apague el led, y cuando llegue a un determinado valor tendra que apagarse, borrar la pantalla lcd y pasar al modo servicio

(Probablemente solo nos haga falta poner la logica en el codigo principal, en vez de usar un thread)
   
## Servicio (service)
   
Dentro de el estado de servicio tendremos dos estados distintos, el de la deteccion de la presencia y el de monitorizacion y productos. Aqui conviene proteger las transiciones en los estados.
1) Deteccion presencia (detection): En este estado tendremos un thread que acceda al ultrasonidos y guarde el valor de la distancia medida. Por otro lado en el bucle principal, se comprobara el valor y si es menor o igual que 1 metro transicionamos al estado de monitorizacion
4) Monitorizacion y productos (monitorice):
   - Temperatura y Humedad (th): En primer lugar tenemos que usar un thread que vaya leyendo el valor de la temperatura y de la humedad cada segundo, y la vaya imprimiendo por la pantalla lcd. Cuando pasen 5 segundos, pasara a la navegacion y seleccion
   - Navegacion y seleccion (select): Tendremos que crear una tarea que este leyendo los valores del joystic y decida si el movimiento es hacia arriba hacia abajo o esta en el medio. Por otro lado en el hilo principal comprobaremos esas acciones y luego mediante un diccionario distinguir el producto que se quiere mostrar, tendremos un contador que indique la fila del producto que queremos, si vamos hacia arriba se restara un valor y si vamos hacia abajo se sumara un valor. Para confirmar la seleccion utilizaremos una interrupcion que con el boton que indique que se ha seleccionado un producto. Es muy conveniente proteger las partes donde manejamos el contador de seleccion para que no haya problemas de seleccion. Cuando se pulse el boton pasaremos al estado de preparacion de la bebida, y guradaremos el valor de la seleccion
   - Preparacion de la bebida (prepare): aqui vamos a imprimir el mensaje de "Preparando {seleccion anterior} ..." durante un tiempo aleatorio entre 4 y 8 segundos, mientras tanto tendremos una tarea que se encargara de aumentar el brillo del led verde, con modulacion PWM, el incremento debe estar escalado al valor del tiempo aleatorio. Despues mostrara "Retire bebida en la pantalla durante 3 segundos y volverá al estado de deteccion.
   - Cancelacion y reinicio: utilizaremos dos interrupciones mediante un boton, una recogera el tiempo en el que recibe un flanco de subida y la otra cuando coge el flanco de bajada, por otro lado en una tarea, estaremos comprobando continuamente el valor de la resta entre los tiempos, si esta entre 2 y 3 segundos, volveremos al estado de detccion de nuevo. Si el valor es menor que 2 o esta entre 3 y 5 segundos, no hacemos nada.
  
## Administracion (admin)

Se accede si pulsamos el boton externo al menos 5 segundos. Si ocurre esto en cualquier fase pasamos al estado admin. Mientras estemos en este estado los LED deben estar encendidos.

De nuevo tendremos que usar los valores que nos devuelve la tarea del joystick para movernos a traves de un menu con distintas opciones. Es conveniente hacer una funcion que devuelva un array en funcion de los valores del joystick, [vert_move, hor_move], de tal manera que vert_move puede ser u (up), n (nothing) o d (down) y los hor_move pueden ser r (rigth) o n (nothing). Nos movemos en el menu hacia arriba o hacia abajo, y podemos ir para atras moviendo a la izquierda. Por tanton tendremos que estar todo el rato chequeando para salir de una opcion del menu si movemos a la izquierda el joystick. Aunque en realidad solo comprobamos eso si han seleccionado una opcion del menu, luego podriamos tener un thread para arriba y abajo en el menu, y un thread para el izquierda en alguna opcion del menu.

Opciones:
1) Ver temperatura (temp): tendremos que crear una tarea que obtenga el valor del la temperatura y la humedad cada segundo e ir refrescando esos valores en la pantalla lcd
2) Ver distancia sensor (distance): hacemos como en el anterior per mostrando la distancia del sensor de ultrasonidos
3) Ver contador (time): Aqui lo que vamos a mostrar es el tiempo que llevamos de ejecucion en segundos, se debe ir refrescando cada segundo. Podemos crear una funcion que lo calcule cuando entramos por primera vez y luego ir encrementando y mostrando el valor en cada segundo
4) Modificar precios (prices): 


<img width="2876" height="2500" alt="image" src="https://github.com/user-attachments/assets/dd9a6c3c-307a-465e-b40f-33f79ce49572" />
