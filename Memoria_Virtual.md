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