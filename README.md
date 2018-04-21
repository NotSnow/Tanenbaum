# Tanenbaum
Resumen de Tanenbaum.

# Resumen Resumido
* **Sistema Operativo**: Capa de software cuyo objetivo es proporcionar a los programas de usuario un modelo de computadora mejor, más simple y pulcro, así como encargarse de la administración de todos los recursos (memoria principal, discos, impresoras...). Es la capa entre el hardware y las aplicaciones de usuario.
* **Modo Kernel**: Tiene acceso completo a todo el hardware y puede ejecutar cualquier instrucción que la máquina sea capaz de ejecutar.
* **Modo Usuario**: Sólo un subconjunto de las instrucciones de máquina es permitido (no se permiten E/S o de control de máquina).
* **Llamada al Sistema**: Instrucción que permite al programa de usuario obtener los servicios del SO (para cambiar entre modo usuario y root se usa la instrucción 'trap').

* **Procesador**: Es capaz de ejecutar un repertorio de instrucciones ensamblador. El SO debe controlar completamente su estado y administrar su uso. Propiedades que pueden tener:
    * **Multihilamiento**: Capazidad de alternar en un tiempo de nanosegundos entre procesos para ofrecer la sensación de poder hacer dos cosas a la vez.
    * **Multinúcleo**: Chips de CPU con más de un procesador en su interior, lo cual permite hacer realmente varias cosas a la vez.

* **Memoria**: No se puede tener un acceso rápido a ella y que tengan una gran capacidad en un mismo dispositivo. Por tiempo de acceso creciente tendríamos Registros, Caché, Memoria Principal (RAM) y Disco. Es en la Memoria Secundaria (Disco) donde se almacena el Sistema de Archivos y la única memoria que permanece al apagar la máquina (memoria no volátil).
* **Dispositivos E/S**: Para cada controlador de dispositivo de E/S debe haber un driver (software de comunicación con el controlador) que se ejecuta en modo kernel y puede cargarse de forma dinámica.
    * **Operaciones de E/S**: espera ocupada (fuerza a la CPU a estar sondeando los diversos controladores).
    * **Interrupciones**: Permite que los controladores manden una señal a la CPu cuando ocurre algún evento en ellos.
    * **DMA**: (Direct Memory Access) que permite que la CPU le mande una orden de movimiento de datos entre memorias y que ésta siga con otros cálculos hasta que el movimiento termine.

* **Proceso**: Programa en ejecución, el cual tiene toda la información necesaria para su ejecución y un espacio de direcciones, que es una lista de ubicaciones de memoria que va desde un mínimo hasta cierto valor máximo, donde el proceso puede escribir y leer información, y donde están contenidos el programa ejecutable, los datos del programa y su pila.
* **Shell**: Interfaz del usuario con el SO que no pertenece a éste. Permite el uso de comodines para sustituir partes de sus cadenas de texto.