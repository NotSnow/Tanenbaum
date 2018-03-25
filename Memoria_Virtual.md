# CONCEPTOS IMPORTANTES
Un computador tiene cierta memoria principal que utiliza para mantener los programas en ejecución. En los computadores modernos se pueden colocar varios programas en memoria al mismo tiempo.

En el caso más simple, la máxima cantidad de espacio de direcciones que tiene un proceso es menor que la memoria principal.

Hoy en día existe una técnica llamada **Memoria Virtual** en la que el SO mantiene una parte del espacio de direcciones en memoria principal y otra parte en disco

## Memoria virtual
La memoria virtual proporciona la habilidad de ejecutar programas más extensos que en la memoria física de la computadora (que la RAM), trayendo pedazos entre la RAM y el disco. También permite ligar dinámicamente bibliotecas en tiempo de ejecución.

## Llamadas al sistema
FUNCIONES DEL SISTEMA OPERATIVO

* Proveer abstracciones a los programas de usuario
* Administrar recursos de la computadora

La **llamada al sitema** generalmente es el mecanismo usado por una aplicación para solicitar un servicio al sistema operatuvo. Normalmente usan una instrucción especial de la CPU.

Cuando una llamada al sistema es invocada, la ejecución del programa es interrumpida y sus datos son guardados para poder ejecutarse luego.

Para hacer más entendible el mecanismo de llamadas al sistema, vamos a echar un vistazo a la llamada **_read_**

```
cuenta = read(fd, buffer, nbytes);
```
La llamada devuelve le nº de bbytes que se leen en _cuenta_ (por lo general el valor de _cuenta_ es el mismo que u_nbytes_ pero puede ser menor si se encuentra el fin del archivo al estar leyendo).

Si hay un error _cuenta_ se establece a '-1' y se coloca el error en _'errno'_

Centrandonos más en el ejemplo... https://image.ibb.co/mS9jLS/92806adb4cbb46e4fa8a34d0aab48641.png

1. Se pasan los parámetros de la función como si fuera una pila (de ahí que estén en orden inversa). Además el segundo parámetro se pasa como referencia, al contrario que los otros, que se pasan como valor. Por esa razón hay que indicar 'contenido de bufer' con el &.
2. Lo hace con buffer.
3. Lo hace con fd.
4. Llamada al procedimiento de biblioteca (típica llamada a procedimiento)
5. La biblioteca coloca por lo general el nº de la llamada al sistema en un lugar en el que el SO lo espera, como por ejemplo, y, en este caso, un registro.
6. Se ejecuta un TRAP para cambiar a modo kernel. Aunque la instrucción TRAP no salta a una dirección donde se encuentre el procedimiento.
7. Por lo que salta a una dirección arbitraria donde hay un campo de 8 bits que proporciona un índice a una tabla en memoria que le indica el salto que tiene que hacer para continuar con el procedimiento.
8. Se le proporciona el índice al manejador de llamadas para devolver el procedimiento a la biblioteca.
9. Ejecuta el manejador de llamadas que le devuelve al procedimiento de biblioteca (otra vez espacio usuario).
10. Por último regresa en la forma usual a la que regresan las llamadas a procedimientos.

Por último el programa usuario limpia la pila.

# Preludio
El trabajo del SO es abstraer una jerarquía de memoria en un modelo útil y administrarla; la parte que hace esto se llama **Administrador de memoria**. En este capítulo nos concentraremos en el modeo del programador de la memoria principal y cómo se puede administrar.

## Sin Abstracción de Memoria
Cada programa veía simplemente la memoria física. Cuando un programa veía la instrucción 
```
MOV REGISTRO1, 1000
```
la computadora movía el contenido **_100_** a **_REGISTRO1_** dentro de la memoria física.

De esta forma no había manera de tener 2 programas ejecutándose a la vez. Ya que se borrarían mutuamente si intentaran acceder a la misma dirección mientras están en ejecución.

Los modelos (a) y (c) tienen la desventaja de que un error en el programa de usuario puede borrar el SO (al estar en memoria de escritura):

https://image.ibb.co/ehA8X7/36018ca4b77ff26c4b4136f29e151c18.png

## Ejecución de múltiple programas sin abstracción
El programa guarda todo el contenido de la memoria en un archivo en disco, más tarde cuando vuelve al programa lo vuelve a traer.

Sin embargo esto trae un problema como se puede comprobar en la imagen...

https://image.ibb.co/bAMwkS/sin_abs.png

El primero de los programas tiene una llave de memoria diferente al segundo para poder distinguirlos. Al principio el programa salta a la dirección 24; el segundo salta a la 28. Cuando los dos programas se cargan consecutivamente en memoria empezando en dirección 0.

Con esto, los programas no se interfieren porque tienen distinta llave, pero las instrucciones (por ejemplo en los JMP) saltan a direcciones que no deberían saltar. Eso es porque los programas se están refiriendo a memoria física.

**Una solución** sería utilizar la **reubicación estática** que consiste en sumarle el valor corresponiente a la posicion en la en disco que se le asocia. Es decir, si un programa se cargaba en la dirección 16,384, se sumaba ese mismo valor.

## Una Abstracción de memoria: Espacios de direcciones
Una solución para permitir que haya varias aplicaciones en memoria al mismo tiempo es la usada anteriormente, con la llave que indica qué instrucción pertenece a cada programa.

La otra es crear la abstracción del espacio de direcciones.

Un **espacio de direcciones** es el conjunto de direcciones que puede utilizar un proceso para direccionar la memoria. Cada proceso tiene su propio espacio de direcciones. Lo complicado es proporcionar a cada programa su espacio de manera que la dirección 28 de un programa indique la ubicación física distinta de la 28 en otro programa.

### Registros base y límite
La solución sencilla utiliza una versión simple de **reubicación dinámica**. Asocia el espacio de direcciones de cada proceso sobre una parte distinta de la memoria física. Esta solución contiene dos registros de hardware especiales **base** y **límite**. Cuando se utilizan estos registros, van aumentando, es decir, van acotando los límites que pueden tener los demás programas para evitar espacios usados.

### Intercambio
Si la memoria física de la computadora es lo bastante grande para contener todos los procesos, los esquemas descritos hasta ahora funcionarán correcramente. Pero en la práctica la RAM es un recurso muy preciado por todos los programas.

La estrategia que se ha desarrollado con los años consiste en utilizar el **intercambio** que consiste en **llevar cada proceso completo a memoria, ejecutarlo durante un tiempo y regresarlo a disco**.

La otra estrategia es conocida como **MEMORIA VIRTUAL**

La operación de intercambio se describe visualmente así: https://image.ibb.co/hgLmKn/intercambio.png

## Administración de memoria libre

### Administración con mapas de bits
La memoria se divide en unidades de asignación (pueden ser del tamaño que convenga), en cada unidad hay un bit correspondiente en el mapa de bits (0 libre y 1 ocupada)

https://image.ibb.co/bs70en/mapa_bits.png

### Administración con listas ligadas

# 1. Introducción
El método llamado Memoria Virtual se basa en la idea de dividir programas en sobrepuestos (_Overlays_) de tal forma que se mantienen en disco y se intercambian hacia dentro de la memoria.

## Aplicando el concepto a Memoria Virtual
Cada programa tiene su espacio de direcciones, que se divide en trozos llamados **páginas**.

Cada página es un rango continuo de direcciones. Las páginas se asocian a la memoria física (aunque no es necesario estar en la memoria física para ejecutar el programa).

Con la **memoria virtual** todo el espacio de direcciones (incluido datos y texto) se puede asociar a la memoria física.

```
Memoria Física = RAM
```

# 2. Paginación
Cuando un programa ejecuta una instrucción como **_MOV REG, 100_** copia el contenido de la dirección de memoria **_1000_** a **_REG_** generando así direcciones virtuales.

Estas direcciones virtuales generadas por el programa se conocen como **direcciones virtuales** y forman el **espacio de direcciones virtuales**. (Si no hubiera memoria virtual, se colocaría directamente la memoria física en el bus de memoria y se lee/escribe directamente en esa dirección).

Cuando se utiliza una memoria virtual, las direcciones virtuales no van directamente al bus de memoria: van al **MMU** (_Memory Management Unit_) que asocia direcciones virtuales a direcciones físicas.

```
Direcciones Virtuales de 16 bits --> hasta 64KB
Memoria Física --> 32KB

El espacio de direcciones VIRTUALES se divide en PÁGINAS --> 4KB
El espacio de direcciones FÍSICAS se divide en MARCOS DE PÁGINA --> 4KB
    *PAGINA y MARCO DE PÁGINA tienen mismo TAMAÑO

Por lo tanto obtenemos 16 PÁGINAS y 8 MARCOS DE PÁGINA
```

Imagen: http://pichoster.net/images/2018/03/22/959e1d8744fd937fdcdb2fa2fc830dea.png


Cuando el programa ejecuta **_MOV REG,0_** ... trata de acceder a la dirección virtual 0. Esta dirección virtual se envía a la MMU. La MMU ve que está en la página 0 (de 0-4095), que esta ya asociada al marco de página 2 (de 8192-12287). Por lo que el bus recibe la dirección **8192**.

Como se puede comprobar en la imagen, sólo 8 de las páginas virtuales se asocian a la memoria física. En hardware, existe un **bit de presente/ausente** que lleva el registro de las páginas presentes en Memoria Virtual.

## Fallo de página
Cuando el programa ejecuta **_MOV REG,32780_** está accediento a un byte dentro de la página virtual 8.

La MMU detecta que la página no está asociada y hace que la CPU haga un **trap** (**_fallo de página_**). El SO selecciona un marco de página que se utilize poco y escribe su contenido de vuelta al disco, obtiene la página y reinicia la instrucción del trap.