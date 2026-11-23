# Maquina expendedora 

## Introduccion
Para esta practica hemos diseñado un software y un hardware para simular una maquina expededora, para ello hemos creado diferentes archivos el archivo vendingMachine.ino, que sera nuestro hilo principal. Por otro lado tenemos la carpeta src que contiene varios archivos:
- config.hpp: contiene las constantes de los pines de la placa arduino
- hardware.hpp/.cpp: contiene los objetos de los sensores hardware que necesitamos en funciones de otros archivos, como el lcd y el dht
- sensors.hpp/.cpp: contiene las variables que se van a compartir entre el hilo principal y los threads, de tal manera que ambos pueden modificar y leer sus valores
- threads.hpp/.cpp: contiene las funciones a las que llamaran los threads del hilo principal, a continuacion detallaremos mejor como funciona cada una.

## Funciones de los threads
En los archivos thread.hpp y threads.cpp podemos encontrar varias funciones distintas, que se llamaran a traves de los threads del hilo principal, cada una de ellas se encarga gestionar valores de algun sensor o valores que modifique una interrupcion. A continuacion explicaremos que hace cada una de ellas

### reset_or_admin_check()
Esta funcion es una de las mas importantes ya que es la encargada de calcular el tiempo que ha pasado desde que se pulso el boton auxiliar que se encarga de pasar de admin a servicio y viceversa, para ello tenemos que hacer uso de dos variables last_inter_global, que se encarga de guardar el tiempo de la ultima interrupcion, y last_val, que se encarga de guardar el valor del boton (HIGH/LOW) en la ultima interrupcion. La interrupcion se ejecuta cada vez que hay un flanco de subida o de bajada.

Una vez dentro lo que haremos es calcular el tiempo que ha pasado desde la ultima interrupcion, si es mayor que 30 ms la consideraremos como valida, si es menor se considerara un rebote. Si es una pulsacion valida, comprobamos si es un flanco de subida o de bajada en ufuncion de last_val. Nosotros tenemos que detectar un flanco de subida primero y luego uno de bajada validos, por eso cuando detectamos el flaco de subida valido le asignamos el last_inter_global y ponemos a true la variable pressed, para que asi cuando capture el falnco de bajada valido, le asignemos el valor de last_inter_global, calculamos la diferencia de tiempo entre ambos flancos y la guardamos en press_time que la gestiona el hilo principal, por ultimo, ponemos pressed a false para que en la proxima llamada no se cumpla esta condicion.

### joystick_pressed()
Esta funcion es parecida a la anterior, sirve para evitar los rebotes del boton en el joystick. En este caso la interrupcion solo se capta cuando tenemos un flanco de bajada, y lo que hace es guardar el tiempo en last_inter_select y poner la variable need2select a true, esto sirve para indicar que se ha hecho una pulsacion y tenemos que esperar a que se hayan omitido los rebotes, una vez que se han omitido, ponemos selected a true para que lo gestione el hilo principal y need2select a flase para que no se vuelva a poner selected a true hasta la siguiente interrupcion

### distance()
Esta funcion es la encargada de calcular la distancia a traves del sensor ultrasonidos. Para ello tenemos que emitir la onda durante 10 microsegundos y dejar de emitir. Despues lo que haremos es esperar al flanco de subida del echo capturara el tiempo, esperar al flanco de bajada de echo y calculara la diferencia de tiempo entre ambas para obtener el tiempo de vuelo, por ultimo, calculamos la distancia con lo operacion (tiempo de vuelo * velocidad del sonido en el aire) / 2. Es valor se quedara guardado en una variable a la que podra acceder el hilo principal

### up_down()
Es la encargada de leer VY del joystick. Si el valor es menor que el umbral de UP y la ultima iteracion estaba en el centro, guardamos en vert_action la accion UP, si el valor es mayor que el umbral de DOWN y la ultima vez estabamos en el centro, guaradamos la accion DOWN, si esta en el rango que se considera centro entonces la accion es accion es STAY y ponemos que estabamos en el centro y si no se cumple ninguna la accion es STAY para no movernos.

### backwards()
Es parecida a la anterior per en este caso lee el valor de VX del joystick, si el valor estaba en el centro y el valor es menor que el umbral de LEFT, decimos que go_backwards, es true (la usaremos en el hilo principal) y que no estamos en el centro. Si es estamos en el rango del centro, no volvemos para atras y estabamos en el centro y si no se cumple ninguna no vamos hacia atras.

### read_dht()
Es una funcion muy sencilla que guarda en dos variables que se usan en el hilo principal, la temperatura y la humedad

## Hilo principal
Nuestro hilo principal consiste en 3 estados distintos START, SERVICE y ADMIN, y cada uno de ellos realizara unas acciones pero antes debemos comentar algunas funciones auxiliares que se usaran mas tarde en loop()

### Funciones auxiliares
#### modify_menu(int mode) 
Esta funcion es la que se encarga de aplicar la accion que modifica up_down. Tenemos un parametro mode para hacer una cosa u otra en funcion de lo que queremos modificar. 
- SERVICE: En service calculamos cual es el siguiente inidice del menu del servicio, sumandole el valor de la accion (UP, DOWN o STAY), si el valor esta en el un rango valido actualizamos el inidice de servicio, mostramos el menu de servico, y actualizamos el ultimo valor del inidce.
- ADMIN: En admin calculamos cual es el siguiente inidice del menu del administrador sumandole al actual el valor de la accion, si el nuevo inidce esta dentro del rango permitido actualizamos el indice del menu de administrador, mostramos el menu del administrador y actualizamos el valor del ultimo inidice
- MODIFY_PRICES: En este caso, lo que hacemos es distinto, calculamos el siguiente valor del precio, le sumamos 0.05 si la accion era UP o le restamos 0.05 si era DOWN, si era STAY le asignamos el valor de modified_price. Despues comprobamos que el nuevo valor sea mayor o igual que cero, si se cumple actualizamos modified_price con el nuevo valor. Enseñamos la modificacion y actualizamos el ultimo precio con el valor.

Cuando acaba actualizamos la accion con el valor STAY para que no este bajando hasta que se vuelva a ejecutar el thread del joystick.

#### show_serv_menu(int op, int last_op)
Con esta funcion imprimiremos por la pantalla lcd si y solo si el estado anterior es distinto al actual, para que la patalla no se este refrescando continuamente. Utilizanod op seleccionamos en un array donde estan los setrings de cada producto y otro array donde tenemos el precio, de tal forma que el indice del producto coincide con el indice de su precio, imprimiendo el valor del producto arriba y el valor del precio abajo.

#### show_ad_menu()
En este caso trabajamos directamente con el valor del indice del menu y su ultimo inidice, en funcion del valor del inidce imprimiremos un mensaje un mensaje u otro, si y solo si los valores del ultimo indice y el actual son distintos. 

#### show_price()
Si los valores del antiguo precio y el actual son distintos entoncees imprimo por pantalla el precio 

#### check_reset_or_admin()
Esta funcion es la que se encargara de gestionar el valor de press_time que modificaba el thread que comente antes. Si estabamos en service y se ha presionado durante un tiempo entre 2 y 3 segundo entonces paso al estado de deteccion y aplico las configuraciones necesarias. Si presiono en el boton mas de 5 segundos y estoy en SERVICE, paso a ADMIN y configuro el estado, y si estaba en ADMIN al reves. Por ultimo, reseteo los valores del tiempo presionado, el tiempo del flanco de subida y bajada validos.

### Setup 
En setup establecemos las configuraciones necesarias antes de empezar la ejecucion. Voy a destacar el tema de los controladores. Tenemos 4 controladores, admin_ctr controla los threads de distance, de read_dht y de backwards; el controlador move_ctr se encarga de controlar el thread de up_down; global_ctr se encarga de controlar el thread de ambos botones y detection_ctr se encarga de controlar el thread de la deteccion tambien

### Loop
Aqui es donde definimos el mecanismo del cambio de estados. Emepezaremos en START

#### START
Tenemos un contador de parapadeos si esta en el estado inicial imprime cargando. Despues vamos calculando que haya pasado un segundo cada vez que se cumpla, cambiasmos el estado del led, y cuando lleguemos a 6 cambios pasamos al estado de servicio

#### SERVICE
A partir de aqui cada vez que entremos en un nuevo estado, configuraremos lo que necesite y ya seguiremos con su logica. Ahora tenemos dos estados de servicio DETECTION y MONITORICE. En el estado de deteccion esperamos a que llegue el cliente, mostrandolo por pantalla y comprobando si la distancia es menor que un metro llamando a detection_ctr, si se cumple pasamos al estado de monitorizacion. En monitorizacion tenemos distintos estados:
- SHOW_TH: guardamos el tiempo cuando entramos y mostramos la temperatura y la humeda, cuando la diferencia entre millis y el tiempo del inicio sean mayores que 5 segundos entonces pasamos al enseñar el menu.
- SHOW_MENU: En este estado lo que hacemos es comprobar si selected es igual a true, si no lo llamamos al move_ctr y mostramos la opcion del menu de servicio, en cuanto selected es igual a true pasamos al estado de PREPARING.
- PREPARING: En este caso al entrar obtenemos un tiempo aleatorio entre 4 y 8 segundos, mostramos el mensaje de preparando y guardamos el tiempo. Despues comprobaremos si se ha cumplido el tiempo aleatorio, si no ha sido el caso hacemos un analogWrite en el led rojo de tal manera que el valor sea igual al tiempo que lleva entre el tiempo que tiene que estar por el valor maximo del led (255), asi va brillando en funcion del porcentaje que lleve, en cuanto hay pasado el timeout, pasa al estado TAKE_DRINK.
- TAKE_DRINK: Por ultimo imprimimos durante 3 segundos el mensaje de retirar la bebida y en cuanto se pase volvemos al estado de deteccion


### ADMIN
Aqui caando entramos tenemos varios estados nuevos:
- SHOW_MENU: En este caso comprobamos si selected es true, si no lo es llamamos al move_ctr y mostramos el menu del administrador, si selected es true, igualamos el admin_state al valor del inidice del menu. 
- SHOW_TH: En este caso llamamos a admi_ctr, para que el read_th_thread lea la temperatura y la humedad y la muestre en el lcd. Ademas tambien corre el go_backwards_thread y en caso de que go_backwards sea true vuelve a SHOW_MENU, esto se realizara igual en los demas estados, por eso no lo comentare en los siguientes.
- SHOW_DISTANCE: Aqui lo que hacemos es llamar a admin_ctr para refrescar el valor de la distancia, la multiplicamos por 100 para pasarla a cm y lo mostramos por la lcd. 
- SHOW_COUNTER: lo que hace es comprobar que haya pasado un segundo desde el ultimo cambio, si es asi se coge el valor de millis, hacer una division entera de mil para pasarlo a segundos, y luego se imprime en la lcd.
- MODIFY_PRICES: Por ultimo en es caso tenemos dos opciones. El estado SHOW_MENU, que llama a move_ctr y se muestra el menu de servicio si selected es false, en caso contrario actualiza el valor del estado con el valor del indice de servicio. Ahora estamos en el defoult, al entrar cogemos el valor del precio en ese inidce y lo guardamos en la variable modified_price, luego a partir de ahi, llamamos a move_ctr y mostramos el precio (mode MODIFY_PRECES), y asi vamos variando su valor con el joystick, esto si selected es false, si es true, lo que hacemos es que el valor del array de precios en el indice sera igual a modified_price y volvemos al menu de servicio, si se detecta go_backwards antes que el selected true, se vuelve al menu sin asignar ningun valor.

Por ultimo destacar que al final del loop, llamamso siempre al controlador que controlaba los threads de los botnes, y llamamos a la funcion que pasaba de un estado o a otro en funcion del tiempo presionado. Por ultimo reseteamos el watchdog.

<img width="2876" height="2500" alt="image" src="https://github.com/user-attachments/assets/dd9a6c3c-307a-465e-b40f-33f79ce49572" />
