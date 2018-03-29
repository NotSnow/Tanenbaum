# Introducción
Con frecuencia los procesos necesitan comunicarse entre sí (Ej: el **pipe** de shell). El resumen de este tema van a ser estos 3 cuestiones:

1. Cómo un proceso puede **pasar información** a otro.
2. Cómo **evitar interferencias**.
3. Garantizar **el orden correcto**.

Cabe destacar que, en el caso de los hilos, pasar información es más sencillo, ya que comparten espacio de direcciones. Sin embargo, sigue siendo complicado evitar que entren en conflicto por la información.

## Condiciones de Carrera
Los procesos pueden compartir espacio de almacenamiento en el que pueden leer y escribir datos (memoria principal, archivo compartido...). Una vez sabido esto vamos a ver un ejemplo en concreto:

NOTA: Este ejemplo es **MUY IMPORTANTE** para entender las carreras criticas
```
Tenemos un spooler de impresión.

Varios procesos van indroduciendo nombres de archivos a imprimir en un spooler.

Un demonio de impresión va comprobando periódicamente si hay archivos para imprimir.

El directorio spooler tiene muchas ranuras numeradas, en cada una con el nombre del archivo.
    - Tiene una variable que indica el siguiente archivo a imprimir. (Imaginemos que es 4)
    - Tiene otra variable LIBRE que indica la siguiente ranura libre. (Imaginemos que es 7)


De manera simultánea, los procesos A y B deciden poner en cola un archivo.

El proceso A lee la variable LIBRE y ve que es 7, justo antes de insertar el archivo y actualizar la variable, se produce una Interrupción de Reloj y el proceso queda bloqueado.

Cuando el proceso A vuelva a la normalidad seguirá con su código y hará lo siguiente que tenía pensado hacer... insertar archivo y actualizar variable.

Sin embargo, durante la Interrupción de Reloj, la CPU puso el proceso B activo y consiguió completar su tarea completa (a diferencia de A), es decir, leyó la ranura disponible, que LIBRE seguía siendo 7 (ya que no le dio tiempo a actualizar a A porque no completó su tarea), insertó el archivo en la ranura 7, y seteó la variable LIBRE en 7+1, o sea en 8.

Por lo tanto, cuando volvió el proceso A a su tarea... tendrá su variable LIBRE errónea.
```

Otros ejemplos de carrera...

```
int suma = 0;

void func_suma(int ni, int nf)
{
    int j;
    for (j = ni; j <= nf; j++)
    {
        suma = suma + j;
    }
}
```

En este último ejemplo, la función **accede todo el rato a una variable global** y la modifica en cada iteración, lo que provoca que, en el caso de que la CPU lo interrumpa, se quede en el medio del bucle con la variable sin actualizar completamente.

Aquí veremos un pequeño arreglo que puede solucionar este último caso. En vez de modificar en cada iteración la variable global, únicamente **actualiza la variable global cuando tiene el problema resuelto con variables locales**:

```
int suma = 0;

void func_suma(int ni, int nf)
{
    int j;
    int suma_parcial = 0;
    for (j = ni; j >=nf; j++)
    {
        suma_parcial = suma_parcial + j;
    }
    suma = suma + suma_parcial;
}
```

# Condiciones para Asegurar Carreras Críticas
Cuando el proceso accede a la memoria compartida o a archivos compartidos. Esta parte de los programas se conoce como **Región Crítica** o **Sección Crítica**.

Necesitamos **4 condiciones** para obtener la solución:
1. **No** puede haber **más de un proceso** dentro de sus regiones críticas.
2. **No** se pueden hacer suposiciones sobr e la **velocidad o número de CPU** (es muy relativo).
3. Ningún proceso que se encuentre fuera de la región crítica puede **bloquear otros procesos**.
4. Ningún proceso debe **esperar indefinidamente** para entrar en su región crítica

En resumen, el comportamiento que deseamos está descrito en la siguiente imagen:

https://image.ibb.co/kRL35S/carrera_critica.png

## Espera Ocupada
Esta técnica se basa en obtener la **Exclusión Mútua** deshabilitando interrupciones.

* Proceso **entra** en Región Crítica --> **DESHABILITA INTERRUPCIONES**.
* Proceso **sale** de Región Crítica --> **HABILITA INTERRUPCIONES** de nuevo.

No es conveniente dar poder a un proceso de usuario para desactivar interrupciones. Estos procesos son inestables y puede que no lleguen a reactivarlas nunca.

Por otro lado, **no funciona en sistemas multicore** ya que sólo deshabilitan las interrupciones en una CPU.

## Variables Candado
Busquemos una solución de Software.

Tenemos una sóla variable compartida de **candado**. Inicialmente está a 0

El proceso 1 evalúa el candado...
* Si _Candado = 0_ --> fija _Candado = 1_ y **entra en Región**.
* Si _Candado = 1_ --> **espera**

```
Es decir, si el Candado es 0 es que no hay nadie en la Región, si Candado es 1 la Región está ocupada.
```
Esta solución tiene el **mismo problema que el spooler**.

## Alternancia Estricta
