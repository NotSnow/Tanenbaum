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
Esta técnica se entiende muy bien con código en C y luego una explicación...

```
proceso_0 ()
{
    while (TRUE)
    {
        while (turno != 0);     // Si no es su turno, se queda evaluando de forma continua la variable turno. A esto se le llama ESPERA OCUPADA
        region_critica();       // Cuando finalmente es su turno (turno = 0) entra en la región crítica (no puede haber nadie dentro ya que al establacer la variable en 0 sólo él, el proceso 0 puede entrar) y hasta que no vuelve a cambiar él mismo la variable no vuelve a entrar nadie.
        turno = 1;              // Sale de la región crítica y pone la variable en 1, es después de ejecutar esta instrucción que deja al proceso 1 entrar.
        region_no_critica();
    }
}
```
```
proceso_1 ()
{
    while (TRUE)
    {
        while (turno != 1);
        region_critica();
        turno = 0;
        region_no_critica();
    }
}

// Como podemos ver, el código de los procesos es idéntico.
```

Hay varias cosas a destacar:
* La acción de estar evaluando constantemente un valor hasta que cambie se le llama **espera ocupada** o **espera activa** y no suel ser positivo ya que **Los procesos trabajan a un ritmo más lento**.
* No se cumple la **condición 3**: se bloquean sin estar en una región crítica.
* Adicionalmente cabe destacar que un candado que usa Espera Activa se llama **Candado de Giro**.

## Solución de Peterson
Es una solución que combina la idea de tomar turnos con los variables candado y las variables de advertencia.

Como en el último caso vamos a ver código junto a una explicación.

```
#define FALSE   0
#define TRUE    1
#define N       2                       // Numero de procesos

int turno;                              // Variable para indicar el turno 
int interesado[N];                      // Al principio all = FALSE. Variable de advertencia para indicar qué procesos están interesados

void entrar_region (proceso)
{
    int otro;                           // Variable auxiliar para almacenar localmente el otro proceso del que se trabaja ahora mismo

    otro = 1 - proceso;                 // Se asocia el otro proceso con el que se trabaja
    interesado[proceso] = TRUE;         // Indico que YO ESTOY INTERESADO
    turno = proceso;                    // Digo que es MI TURNO
    while (turno == proceso && interesado[otro] == TRUE)        // La primera condición siempre va a ser TRUE, la segunda sirve para decirsi el otro proceso interesado[otro] = TRUE o al revés interesado[otro] = FALSE
                                                                // El while se queda en ESPERA si MI TURNO es TRUE (que lo es) y el otro proceso está interesado, es decir si (TRUE && TRUE)
                                                                // Sin embargo, se salta el while y entra en la región crítica si es MI TURNO y el otro proceso no esta interesado, es decir, salió de su región y estableció su pripio interesado[] = FALSE
}

void salir_region (proceso)
{
    interesado[proceso] = FALSE;        // Aquí está la clave, donde este proceso deja claro que no esta interesado en la región crítica y permita pasar del while cuando lo ejecute el otro proceso
}

// Por si te queda la duda, al principio los 2 interesado[N] se establecen en FALSE, por lo que cuando el primer proceso ejecute la función para entrar en la región, el bucle será (TRUE && FALSE) y pasará de él, y entrará en la región. 

// El otro proceso se quedará en ese bucle (TRUE && TRUE) esperando a que el que está dentro salga y establezca su FALSE y así poder pasar a (TRUE && FALSE) y así sucesivamente.
```

## La instrucción TSL
Esta técnica requiere un poco de ayuda del hardware, ya que usa la instrucción atómica TSL.

Esta instrucción lee el contenido de la palabra de memoria **_candado_** y lo guarda en **_reg_**, y después asigna un valor distinto de 0 de nuevo en **_candado_**. Esto es muy parecido a la variable candado de la que habíamos hablado hace unos puntos atrás, sin embargo, TSL nos proporciona seguridad e indivisibilidad a la hora de hacer las operaciones de **leer y almacenar** la palabra.

La CPU ejecuta TSL y bloquea el bus de memoria impidiendo a otras CPU el acceso a memoria. Se podría pensar que no es buena idea bloquear el acceso, sin embargo, al ser una **instrucción atómica** se garantiza que se ejecute siempre entera y al impedir a otras CPU acceder a memoria es muy distinto de deshabilitar las interrupciones. Ya que si deshabilitamos las instrucciones, las demás CPUs pueden acceder a la memoria igualmente.

Veamos un ejemplo de código, aunque no es realmente importante al ser a muy bajo nivel.

```
entrar_region:
    TSL reg, candado    // Se copia candado a registro y se fija candado a distinto de 0
    EQ reg, #0          // Se compara registro con 0
    JNE entrar_region   // Si no es 0 se vuelve a entrar en bucle
    RET                 // Cuando sea reg igual a 0 entra en región crítica

salir_region:
    MOVE candado, #0    // Almacena 0 en candado
    RET                 
```

## Dormir y Despertar
Tanto las soluciones de **Peterson** como la solución mediante **TSL** son **válidas**. Pero todas tienen el problema de que recurren a la **Espera Ocupada**.

Además, este método no sólo no desperdicia tiempo de CPU, sino que evita el **problema de inversión de prioridades**: dados dos procesos con prioridad alta (proceso A) y baja (proceso B), el planificador le da siempre prioridad al proceso A. Si B se encuentra en un momento en la región crítica y el proceso A está listo para ejecutarse, la CPU va a intentar ejecutar siempre A. Por lo que B tarda muchísimo en salir de su región crítica.

La solución que se propone aquí es no comprobar si se puede entrar en cada ciclo, sino usar las llamadas al sistema **_Sleep_** y **_WakeUp_**
* **_Sleep_**: el proceso que llama se bloquea.
* **_WakeUp_**: el proceso dentro del parámetro se desbloquea.

Este método se ve en el problema del Productor-Consumidor.

## Problema del Productor-Comsumidor
Primeramente vamos a ver el código para explicarlo y que se vea mejor visualmente.

```
#define N   100                                 // número de huecos en el bufer
int cuenta = 0;                                 // número de elementos actualmente en el bufer

void productor()
{
    int elemento;

    while (TRUE)
    {
        elemento = producit_elemento();         // se produce un elemento
        if (cuenta == N) sleep();               // si el bufer está lleno se bloquea (para no producir más)
        insertar_elemento(elemento);            // si el bufer NO esta lleno se inserta el elemento recientemente producido 
        cuenta += 1;                            // se informa en la variable global que se ha producido 1
        if (cuenta == 1) wakeup(consumidor);    // si existe algún elemento en el bufer preparado para consumir, se despierta al consumidor (esto tiene sentido si el bufer está vacío y el consumidor se ha dormido)
    }
}

void consumidor()
{
    int elemento;
    
    while (TRUE)
    {
        if (cuenta == 0) sleep();               // si no hay elementos en el bufer no puedo consumir y me bloqueo
        elemento = quitar_elemento();           // se consume un elemento
        cuenta -= 1;                            // se informa en la variable global que se ha consumido 1
        if (cuenta == N-1) wakeup(productor);   // si acabo de consumir un elemento estando el bufer lleno, significa que no está lleno y quiero despertar al productor para que vuelva a producir
    }
}
```

Visto el código anterior podemos sacar varias conclusiones:
1. Está bien pensado que el productor se duerma si el bufer está lleno y que sea el consumidor el que lo despierte cuando haya quitado personalmente un elemento del bufer.
2. Está bien pensado que el consumidor se duerma si el bufer está vacío y que sea el productor el que lo despierte cuando haya producido personalmente un elemento al bufer.
3. Sin embargo pensemos... **¿qué pasa con la variable global?** Es decir, ¿y si el productor produce un elemento y en ese momento la CPU le quita el control y se lo pasa al consumidor? en ese caso no le daría tiempo a actualizar la variable global e informar al consumidor que se ha producido un elemento y.

El último punto es de suma importancia, de hecho, lo mismo pasa con el consumidor... ¿y si el consumidor quita un elemento del bufer y no le da tiempo a actualizar el bufer porque la CPU le ha dado control al productor? ¡¡En ese caso el productor tampoco sabe que realmente se ha comido un elemento!!

Por estas ultimas razones se han inventado los semáforos.

## Semáforos
