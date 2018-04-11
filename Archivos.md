# Sistemas de Archivos
## Introducción
Todas las aplicaciones de la computadora requieren almacenar y recuperar información. Mientras un proceso está en ejecución, puede almacenar una cantidad limitada de información dentro de su propio espacio de direcciones. Sin embargo, la capacidad de almacenamiento está restringida por el tamaño del espacio de direcciones virtuales (lo cual puede ser demasiado pequeño).

Por otro lado, cuando un proceso termina, la información que hay en el espacio de direcciones se pierde, por lo que hay que buscar una forma de guardarla adecuadamente.

Un tercer problema es que frecuentemente es necesario que varios procesos accedan a cierta información simultáneamente. La manera de resolver este problema es hacer que la información en sí sea independiente de cualquier proceso.

## Definiciones y Conceptos
* Los **archivos** son unidades lógicas de información creada por los procesos.
* La información que almacena en los archivos debe ser **persistente**, es decir, no debe ser afectada por la creación y terminación de procesos (sólo debe desaparecer cuando el propietario lo elimina de forma explícita).
* Los archivos son administrados por el **Sistema Operativo**. La parte del SO que trata con los archivos se conoce como **sistema de archivos** y es el tema del capítulo.

## Nomenclatura de Archivos
Las reglas exactas para denominar archivos varían un poco de un sistema a otro. Por ejemplo, algunos sistemas de archivos diferencian mayúsculas de minúsculas (como UNIX) y otros no (como MS-DOS).

Por otro lado, Windows 95 y 98 usan el mismo sistema de archivos que MS-DOS conocido como **FAT-16**. Más tarde otros Windows comenzaron a implementar el sistema de archivos **FAT-32**. A partir de Windows NT admiten ambos sistemas de archivos, aunque en realidad ya son obsoletos.

Estos SO tienen un sistema de archivos nativo NTFS con diferentes propiedades.

### Extensiones de Archivos
Muchos SO aceptan nombres de archivos en dos partes, separadas con un punto _main.c_. La parte que va después del punto se conoce como **extensión del archivo** y por lo general indica algo sobre su naturaleza.

En el caso de UNIX las extensiones de archivo son sólo convenciones y no son obligatorias, además, también permite más de una extensión (como _paginaWeb.html.zip_).

En algunos casos las convenciones son muy recomensables de seguir, por ejemplo, algunos compiladores de C leen la extensión del archivo para comprobar si se trata de archivo de código.

## Estructuras de Archivos
Los archivos se pueden estructurar en una de varias formas (como se muestra en la figura)

![Estructuras Archivos](https://image.ibb.co/hO0ZhH/estructurasarchivos.png)

Tanto UNIX como Windows utilizan la **secuencia de bytes sin estructura**: el SO no sabe, ni le importa lo que hay en el archivo (todo lo que ve son bytes). Cualquier significado debe ser impuesto por los programas a nivel usuario (lo cual provee la máxima flexibilidad al permitir que los programas de usuario coloquen cualquier cosa que quieran y lo denominen de la manera más conveniente que vean).

En el modelo 'b' de la imagen, el archivo es una **secuencia de registros**: cuando lee o escribe lo hace sobre uno de los registros. Y si necesita agregar nueva información agrega un registro.

En el modelo 'c' de la imagen muestra un **árbol de registros** (no todos necesariamente de la misma longitud). Cada registro contiene un **campo llave** en una posición fija dentro de cada registro, que el árbol utiliza para ordenar el árbol y realizar búsquedas rápidamente. Aquí para obtener un registro que se quiera, es necesario tener la llave.

## Tipos de Archivos
1. Los **archivos regulares** son los que contienen información de usuario (nos centraremos en estos).
2. Los **directorios** son sistemas de archivos para mantener la estructura del sistema de archivos.
3. Los **archivos especiales de caracteres** se relacionan coon la E/S.
4. Los **archivos especiales de bloques** se utilizan para modelar discos.

### Archivos Regulares
Los Archivos Regulares pueden ser Archivos ASCII o binarios

El contenido de los **Archivos ASCII** son líneas de texto. La ventaja es que si la mayoría de programas usan este tipo de archivos no sólo para expresar texto regular, sino para E/S, es fácil conectar la salida de un programa con la entrada de otro (el pipe de shell poro ejemplo).

El contenido de los **Archivos Binarios** es un contenido incomprensible de caracteres. Por lo general tienen cierta estructura interna conocida para los programas que los utilizan.

## Acceso a Archivos
Los primeros Sistemas Operativos proporcionaban sólo un tipo de acceso: **acceso secuencial** (útil cuando el medio de almacenamiento era cinta magnética en lugar de disco).

Cuando se empezó a usar discos para almacenar archivos, se hizo posible leer registros en distinto orden. Estos archivos se llaman **archivos de acceso aleatorio**.
1. _read_ da la posicion del archivo en la que se va a empezar a leer.
2. _seek_ establece la posición actual donde empezar a leer.

## Atributos de Archivos
Es necesario asociar información adicional a los archivos para que el SO los pueda gestionar mejor. A estos elementos se le llaman **atributos** o **metadatos**

![Atributos](https://image.ibb.co/hOyGUx/atributos.png)

## Operaciones de Archivos
1. _Create_: El archivo se crea sin datos. El propósito de la llamada es anunciar la llegada del archivo y establecer algunos de sus atributos.
2. _Delete_: Cuando el archivo ya no se necesita, se tiene que eliminar para liberar el espacio en disco. Siempre con una llamada al sistema.
3. _Open_: El propósito de esta llamada es permitir que el sistema lleve los atributos y la lista de direcciones de disco a memoria principal para tener acceso rápido a llamadas posteriores.
4. _Close_: Cuando ya no es necesario referirse al archivo en memoria principal se debe cerrar el archivo para liberar espacio en la tabla interna.
5. _Read_: Los datos se leen del archivo. Por lo general, los butes provienen de la posición actual.
6. _Write_: Los datos se escriben en el archivo otra vez, por lo general en la posición actual (si es el final del archivo, este aumenta de tamaño y si es en mitad del archivo se sobresctiben).
7. _Append_: Es una forma restringida de _write_. Sólo puede agregar datos al final del archivo.
8. _Seek_: Para los archivos de acceso aleatorio, se necesita un método para especificar de dónde se van a tomar los datos. Esta llamada reposiciona el apuntador del archivo en una posición específica del archivo.
9. _Get Atribbutes_: Obtiene los atributos del archivo, el programa 'make', por ejemplo, necesita obtener los tiempos de modificación para ver si necesita recompilar o ya está actualizado el ejecutable que genera.
10. _Set Atribbutes_: Algunos atributos puede establecer el usuari y se pueden mofificar. La mayoría de banderas que se aprecia en la última imagen caen en esta categoría.
11. _Rename_: El usuaio cambia el nombre sdel archivo.

Por supuesto existen más **llamadas al sistema** pero estas son las más comunes y merecía la pena echarles un vistazo en este documento.

```
#define TAM_BUF     4096
#define MODO_SALIDA 0700

int main(int argc, char *argv[])
{
    int     ent_da;
    int     sal_da;
    int     leer_cuenta;
    int     escribir_cuenta;
    char    buffer[TAM_BUF];

    if (argc != 3)  exit(1);                    // Error de sintaxis

    ent_da = open(argv[1], O_RDONLY);           // Abrimos el archivo (permisos de lectura)
    if (ent_da < 0) exit(2);
    sal_da = creat(argv[2], MODO_SALIDA);       // Creamos archivo destino (full permisos al owner)
    if (sal_da < 0) exit(3);

    while(TRUE)
    {
        leer_cuenta = read(ent_da, bufer, TAM_BUFER);       // Leemos de 'ent_da' tantos bytes como 'TAM_BUFER' y guardarlos en 'bufer'
        if (leer_cuenta <= 0)   break;
        escribe_cuenta = write(sal_da, bufer, leer_cuenta); // Escribimos en 'sal_da' lo que pone en 'bufer' los bytes que retorne 'leer_cuenta' ('read' devuelve el numero de bytes leidos)
        if (escribir_cuenta <= 0)   exit(4);
    }

    close(ent_da);
    close(sal_da);
    if (lee_cuenta == 0)
    {
        exit(0);
    } else
    {
        exit(5);
    }
}
```


# Directorios
Para llevar el registro de los archivos, los sistemas de archivos por lo general tienen **directorios** o **carpetas**.

## Sistemas de Directorios de Un Nivel
La forma más simple de un sistema de dorectorios es tener un directorio que contenga todos los archivos. Tiene el nombre de **Directorio Raiz** aunque si es el único directorio... el nombre no importa mucho.

![Directorio Un Nivel](https://image.ibb.co/fnQf6c/directoriounnivel.png)


## Sistemas de Directorios Jerárquicos
Para los computadores del S. XXI con miles de archivos es imposible encontrar algo si todos los archivos estuvieran en un sólo directorio.

Lo que se necesita es una jerarquía (un **árbol de directorios**). Además, si varios usuarios comparten un servidor de archivos común, como se da en el caso de muchas redes de empresas, cada usuario puede tener un directorio raíz privado para su propia jerarquía.

![Directorio Jerarquico](https://image.ibb.co/eTO06c/jerarquicas.png)

## Nombres de Rutas
Cada archivo recibe un **nombre de ruta absoluto** qus consiste en la ruta desde el directorio raíz hasta al archivo (/home/Tanenbaum/Escritorio/mailbox)

    En UNIX, los componentes de la ruta van separados por /.
    En Windows, el separador es \.
    En MULTICS era >

Cuando utilizamos el primer caracter del nombre del archivo '/' (en el caso de UNIX) nos estamos refiriendo a una ruta absoluta.

Por otro lado, están los **nombres de ruta relativa**. Los cuales se utilizan dentro del concepto de **directorio de trabajo** o **directorio actual**. Esto quiere decir que, cuando el usuario designa un directorio de trabajo (por ejemplo, entrando mediante 'cd' hasta un directorio como por ejemplo, 'Escritorio'), dentro del directorio de trabajo los nombres de las rutas que no empiezen por el separador se toman en forma relativa.

    cp /home/Tanenbaum/Escritorio/mailbox   /home/Tanenbaum/Escritorio/mailbox.bak
    ó estableciendo 'Escritorio' como directorio de trabajo:
    cp mailbox  mailbox.bak

La mayoría de los SO que proporcionan un sistema de directorios jerárquicos tienen dos entradas especiales en cada directorio: '.' y '..'
* '**.**': Se refiere al directorio actual.
* '**..**': Se refiere a su padre (excepto en el directorio raíz que se refiere a sí mismo).

![Arbol de Directorios](https://image.ibb.co/bCdQ6c/arboldedirectorios.png)
