# Procesos
Todas las computadoras modernas ofrecen varias cosas al mismo tiempo, lo cual, aunque no nos paremos a pensarlo, existe gracias a los distintos procesos.

Por ejemplo, cuando se arranca un PC, se inician muchos procesos de forma secreta que a menudo el usuario desconoce. Uno de ellos puede ser el encargado de esperar el correo electónico entrante o para que el antivirus compruebe periódicamente la disponibilidad de nuevas definiciones de virus. Aunque también existen otros procesos explícitos como arrancar una terminal u otro programa.

La conmutación entre dos procesos es muy rápida. Tanto que a veces existe una sóla CPU que trabaja en 1 segundo con decenas de procesos dando la apariencia de un paralelismo (que en realidad es **pseudoparalelismo**). Para obtener paralelismo real debemos trabajar con más de una CPU al mismo tiempo sobre la memoria física (**Multiprocesadores**).

## El Modelo del Proceso
Los programas se organizan en varios **procesos secuenciales**. Un proceso no es más que la instancia de un programa en ejecución (que incluye sus propios valores de contador, registros y variables). Cada proceso tiene su propia CPU virtual que, en realidad es la misma CPU que conmuta de un proceso a otro. A esta conmutación es a la que se llama **multiprogramación**.

![Contador Flujo Proceso](https://image.ibb.co/kmU71c/contadorflujoproceso.png)

Como podemos comprobar en la imagen existe **un sólo contador de programa físico**, por lo que cuando conmuta de un proceso a otro, se carga el contador lógico de cada programa que se guarda y carga cada vez que se conmuta de proceso.

### Diferencia entre Proceso y Programa
Imaginemos que existe un cocinero (**CPU**) que esta preparando un pastel (**proceso** "preparar pastel"). Como no sabe hornearlo necesita leer los ingredientes den libro (**E/S**). De repente, su hijo se hace una herida y el cocinero va rápidamente a ponerle un vendaje (conmuta a un proceso de mayor prioridad, **proceso** "vendar a su hijo"). La clave es que el programa es en sí la receta pero este programa se puede ejecutar por distintos procesos.

## Creación de un Proceso
Hay cuatro eventos principales que provocan la creación de procesos:
1. El **arranque del sistema**.
2. Que un **proceso de una llamada al sistema** para crear otro proceso.
3. Una **petición de usuario** para crear un proceso.
4. El inicio de un **trabajo por lotes**.

A menudo existen procesos que se suelen crear al arranque del sistema llamados **demonios** que permanecen en segundo plano para manejar ciertas actividades en segundo plano. Otras veces un proceso necesita la ayuda de otro distinto para realizar su trabajo. O el usuario ejecuta un comando en la terminal o inicia un programa haciendo doble click en el entorno de escritorio.

La última situación se aplica a los Sistemas de Procesamiento por Lotes, donde los usuarios pueden enviar trabajos de procesamiento por lotes. Cuando el sistema operativo decide que tiene los recursos para ejecutar otro trabajo, crea un proceso y ejecuta el siguiente trabajo de la cola de entrada.

La única forma que se tiene de crear un proceso es hacer una llamada al sistema y ejecutar un **_fork()_**. Con esta llamada el padre crea un clon exacto del proceso que hizo la llamada. De hecho, al arrancar un sistema UNIX se crea el primer proceso llamado init que es el encargado de crear el primer proceso hijo, así van haciendo continuamente hijos y ampliando todos los procesos del sistema.

Tanto en UNIX como en Windows, una vez que se crea un proceso, el padre y el hijo tienen sus propios espacios de direcciones distintos. Sin embargo, el espacio de direcciones inicial del hijo es una **copia** del padre (pero en definitiva son dos espacios distintos). Aún así, el proceso hijo también comparte otros recursos de su padre como los archivos abiertos.

## Terminación de Procesos
Hay cuatro eventos principales que provocan la terminación de procesos:
1. Salida normal (voluntaria): normalmente es cuando un proceso termina su trabajo, la llamada para terminarse a sí mismo es _exit()_
2. Salida por error (voluntaria): si el proceso descubre que existe un error (como por ejemplo que a un comando se le pasen argumentos inválidos) puede terminarse a sí mismo (o puede decidir que no, es cosa de cómo esté programado).
3. Error fatal (involuntaria): puede ser provocado al dividir entre 0, hacer referencia a una instrucción ilegal o a una parte de memoria inexistente... el proceso no sabe lo que hacer y se interrumpe.
4. Eliminado por otro proceso (involuntaria): si un proceso ejecuta un _kill()_ a otro proceso.

## Jerarquías de Procesos
En UNIX, un proceso y todos sus hijos, junto con sus posteriores descendientes, forman un grupo de procesos. Cuando un usuario envía una señal del teclado, ésta se envía a todos los miembros del grupo de procesos actualmente asociado con el teclado, pero de manera individual, cada proceso puede atrapar la señal, ignorarla, etc.

Por otro lado, en Windows, cuando un proceso crea un hijo recibe un **manejador** que puede usar para controlar al hijo. Este manejador se puede pasar a otros procesos para que lo controlen ellos. Es la principal diferencia de por qué en Windows se invalida la jerarquía de procesos.

## Estados de un Proceso
Cuando un proceso se bloquea, lo hace debido a que por lógica no puede continuar, comúnmente porque está esperando a E/S o a la salida de otro proceso que aún no ha terminado. También es posible que un proceso esté listo para ejecutarse pero el sistema haya decidido asignarle a CPU a otro proceso.

1. **En ejecución** (está usando la CPU en ese instante)
2. **Listo**        (ejecutable, el sistema ha decidido asignar la CPU a otro proceso)
3. **Bloqueado**    (No puede ejecutarse hasta que ocurra cierto evento externo a él)

