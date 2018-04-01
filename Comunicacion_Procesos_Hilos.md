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
1. **No** puede haber **más de un proceso** dentro de sus regiones Críticas.
2. **No** se pueden hacer suposiciones sobr e la **velocidad o número de CPU** (es muy relativo).
3. Ningún proceso que se encuentre fuera de la Región Crítica puede **bloquear otros procesos**.
4. Ningún proceso debe **esperar indefinidamente** para entrar en su Región Crítica

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
        region_critica();       // Cuando finalmente es su turno (turno = 0) entra en la Región Crítica (no puede haber nadie dentro ya que al establacer la variable en 0 sólo él, el proceso 0 puede entrar) y hasta que no vuelve a cambiar él mismo la variable no vuelve a entrar nadie.
        turno = 1;              // Sale de la Región Crítica y pone la variable en 1, es después de ejecutar esta instrucción que deja al proceso 1 entrar.
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
* No se cumple la **condición 3**: se bloquean sin estar en una Región Crítica.
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
                                                                // Sin embargo, se salta el while y entra en la Región Crítica si es MI TURNO y el otro proceso no esta interesado, es decir, salió de su Región y estableció su pripio interesado[] = FALSE
}

void salir_region (proceso)
{
    interesado[proceso] = FALSE;        // Aquí está la clave, donde este proceso deja claro que no esta interesado en la Región Crítica y permita pasar del while cuando lo ejecute el otro proceso
}

// Por si te queda la duda, al principio los 2 interesado[N] se establecen en FALSE, por lo que cuando el primer proceso ejecute la función para entrar en la Región, el bucle será (TRUE && FALSE) y pasará de él, y entrará en la Región. 

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
    RET                 // Cuando sea reg igual a 0 entra en Región Crítica

salir_region:
    MOVE candado, #0    // Almacena 0 en candado
    RET                 
```

## Dormir y Despertar
Tanto las soluciones de **Peterson** como la solución mediante **TSL** son **válidas**. Pero todas tienen el problema de que recurren a la **Espera Ocupada**.

Además, este método no sólo no desperdicia tiempo de CPU, sino que evita el **problema de inversión de prioridades**: dados dos procesos con prioridad alta (proceso A) y baja (proceso B), el planificador le da siempre prioridad al proceso A. Si B se encuentra en un momento en la Región Crítica y el proceso A está listo para ejecutarse, la CPU va a intentar ejecutar siempre A. Por lo que B tarda muchísimo en salir de su Región Crítica.

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
        elemento = producir_elemento();         // se produce un elemento
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
Un semáforo es un TAD (Tipo Abstracto de Datos) que toma valores enteros positivos, se usa para restringir o permitir el acceso a regiones Críticas. Es muy usado actualmente porque funciona en Sistemas de Multiprocesamiento.

A estos semáforos se le asocian unas operaciones:
1. Operación **down**
```
Semáforo = 0        --> El proceso de bloquea.
Semáforo > 0        --> Se decrementa el semáforo 1 unidad.
```

2. Operación **up**
```
Semáforo = 0        --> Se incrementa el semáforo 1 unidad.
Semáforo bloqueado  --> Se desbloquea el proceso y se pone el semáforo a 0.
```

La solución que proponen es usar 3 semáforos
* **_llenas_**: contabiliza el número de ranuras llenas (de elementos).
* **_vacías_**: contabiliza el número de ranuras vacías.
* **_mutex_**:  asegura que el productor y el consumidor no tengan acceso al búfer al mismo tiempo.

Los semáforos que se inizializan a 1 y son utilizados por más de un proceso para controlar que sólo uno pueda estar en la Región Crítica se llaman **semáforos binarios**; en este caso **_mutex_** lo es.

Ahora veamos de nuevo el código con semáforos...

```
#define N   100                                 // número de huecos en el bufer
typedef int semáforo                            // se establece el tipo de dato que es un semáforo
semaforo mutex = 1;                             // semáforo binario para controlar que sólo uno pueda entrar en la region critica
semaforo llenas = 0;                            // número de elementos actualmente en el bufer
semaforo vacias = N                             // número de huecos que hay en el bufer

    /* De momento las úncas direfencias con respecto a la solución anterior es que en vez de tener una variable definida como máximo de elementos y otra con los elementos actuales que se encuentran*/
    /* Tenemos un semaforo que nos indica los huecos (ayudado por la variable definida anteriormente, otro los elementos que están en el bufer y otro semáforo para evitar que entre más de un proceso en la Región Crítica*/                         

void productor()
{
    int elemento;

    while (TRUE)
    {
        elemento = producir_elemento();         // se produce un elemento
        down(&vacias);                          // si se llenara el bufer, vacias es 0, por lo que este down bloquearía el proceso, sino, va bajando la cuenta de vacías...
        down(&mutex);                           // pone el semaforo en 0, ahora si el consumidor lee este semaforo se quedaría bloqueado ya que lo bajaría de 0 a bloqueado.
        insertar_elemento(elemento)             // coloca el elemento estando en la Región Crítica.
        up(&mutex);                             // pone el semáforo a 1 de nuevo, si el consumidor se bloqueó por culpa de intentar entrar mientras estaba en 0, éste se despierta al ejecutar esta línea
        up(&llenas);                            // se incrementa la cuenta de elementos en el bufer (al igual que el anterior up, si el otro proceso estuviera bloqueado por ejecutar un down(&llenas) mientras estaba el bufer vacío, después de esta línea se desbloquearía de nuevo el consumidor)
    }
}

void consumidor()
{
    int elemento;
    
    while (TRUE)
    {
        down(&llenas);                          // si el semaforo estuviera en 0 (es decir, no hubiera ningun elemento que consumir) se bloquearía y tendría que esperar a que el productor ejecute su última línea
        down(&mutex);                           // pone el semaforo en 0, ahora si el consumidor lee este semaforo se quedaría bloqueado ya que lo bajaría de 0 a bloqueado.
        elemento = quitar_elemento()            // quita el elemento del bufer estando en la Región Crítica
        up(&mutex);                             // pone el semáforo a 1 de nuevo, si el productor se bloqueó por culpa de intentar entrar mientras estaba en 0, éste se despierta al ejecutar esta línea
        up(&llenas);                            // se incrementa la cuenta de elementos en el bufer
    }
}
```

## Mutexes
En el último ejemplo se utilizaron semáforos para poder contar los elementos y los huecos, sin embargo, si no fuera necesario llevar la cuenta de nada, se podría utilizar una versión simplificada llamada **mutex**.

Un **mutex** es una variable que puede tomar valores 0 o 1 (abierto o cerrado). Se utilizan únicamente dos procedimientos:
1. Cuando un proceso accede a la Región Crítica --> llama a _**mutex_lock**_
2. Cuando un proceso sale de la Región Crítica  --> llama a _**mutex_unlock**_

Estos mutexes se parecen mucho a los **semáforos binarios**, por ejemplo, en la última sección de código hemos creado un semáforo binario llamado _mutex_ que prácticamente desempeña la misma función que un mutex real. Sin embargo, los mutex puros se **implementan de forma distinta** ya que siempre se usa una instrucción TSL o similar.

## Mutexes en hilos
Si un hilo desea entrar en la Región Crítica, primero trata de cerrar el mutex asociado. Si el mutex está abierto, el hilo puede entrar y se bloquea automáticamente y rápidamente. Si varios hilos tratan de entrar en la Región pero el mutex está cerrado, cuando el hilo que lo estaba ocupando abre el mutex, se selecciona un hilo al azar para entrar. Mientras que los demás vuelven a esperar bloqueados.

Los bloqueos no son obligatorios, es responsabilidad del programador que los hilos se usen de forma correcta. Para ayudar a este usuario se establecieron varias llamadas a mutexes en pthread

|   LLamada de Hilo  |   Descripción |   Comentario    |
| ----------------- |:---------:|:-----------------|
|Pthread_mutex_init|Crea un mutex| simplemente lo crea, lo puede lockear o no|
|Pthread_mutex_destroy|Destruye un mutex|simplemente lo destruye para que no se vuelva a usar|
|Pthread_mutex_lock|Adquiere un mutex o se bloquea|trata de adquirir el mutex y se bloquea si está cerrado|
|Pthread_mutex_trylock|Adquiere un mutex o falla|trata de adquirir el mutex y si falla suelta el error pero permite tener alternativa en el código|
|Pthread_mutex_unlock|Libera un mutex|abre un mutex y libera un hilo al azar si estuviera bloqueado|

### Variables de Condición
Estas variables permiten que los hilos se bloqueen cuando una condición no se cumple. Las variables están asociadas a mutexes.

En el ejemplo de Productor-Consumidor, si el productor descubre que el bufer está lleno se bloquea atómicamente con un mutex **y se queda comprobando en Espera Ocupada** a que pueda entrar.

Por eso estas **Variables de Condición** esperan cambios que den sentido a evaluar nuevamente una condición y asi **evitar la Espera Ocupada**.

|   LLamada de Hilo  |   Descripción  |
| ------------------ |   -----------  |
|Pthread_cond_init|Crea una Var. Condición|
|Pthread_cond_destroy|Destruye una Var. Condición|
|Pthread_cond_wait|Bloquea el hilo que hace la llamada hasta que otro le señala|
|Pthread_cond_signal|Envía una señal a otro hilo y lo despierta|
|Pthread_cond_broadcast|Envía señal a varios hilos y los despierta|

### Problema Productor-Consumidor resuelto con MUTEXES y VARIABLES DE CONDICIÓN

```
#define MAX         1000000000
pthread mutex_t     the_mutex;
pthread cond_t      condc;
pthread cond_t      condp;
int buffer = 0;

void *productor(void *ptr)
{
    for ( i=1; i<=MAX; i++)
    {
        pthread_mutex_lock(&the_mutex);
        while (buffer != 0)
        {
            pthread_cond_wait(&condp, &the_mutex);
        }
        buffer = i;
        pthread_cond_signal(&condc);
        pthread_mutex_unlock(&the_mutex);
    }
    pthread_exit(0);
}


void *consumidor(void *ptr)
{
    for( i=1; i<= MAX; i++)
    {
        pthread_mutex_lock(&the_mutex);
        while (buffer == 0)
        {
            pthread_cond_wait(&condc, &the_mutex);
        }
        buffer = 0;
        pthread_cond_signal(&condp);
        pthread_mutex_unlock(&the_mutex);
    }
    pthread_exit(0);
}


int main()
{
    pthread_t   prod;
    pthread_t   cons;
    
    pthread_mutex_init (&the_mutex, 0);
    pthread_cond_init (&condc, 0);
    pthread_cond_init (&condp, 0);
    pthread_create (&cons, 0, consumidor, 0);
    pthread_create (&prod, 0, productor, 0);
    pthread_join (prod, 0);
    pthread_join (cons, 0);
    pthread_cond_destroy (&condc);
    pthread_cond_destroy (&condp);
    pthread_mutex_destroy (&the_mutex);
}
```