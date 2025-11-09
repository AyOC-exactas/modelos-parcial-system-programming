# Recuperatorio 2do Parcial de Arquitectura y Organización del Computador - 1c2025

## Enunciado
Como muchas actividades son mas divertidas de hacer en parejas, como bailar un tango o jugar al truco, se pensó que las tareas podrían trabajar en conjunto para poder solucionar problemas de forma cooperativa.
Nuestro sistema implementará una nueva funcionalidad llamada _parejas_ con las siguientes características:
- Toda tarea puede pertenecer **como máximo** a una pareja.
- Toda tarea puede _abandonar_ a su pareja.
- Las tareas pueden formar parejas cuantas veces quieran, pero solo pueden pertenecer a una en un momento dado.
- Las tareas de una pareja comparten un area de memoria de 4MB ($2^{22}$ bytes) a partir de la direccion virtual `0xC0C00000`. Esta región se asignará bajo demanda. Acceder a memoria no asignada dentro de la región reservará sólo las páginas necesarias para cumplir ese acceso. Al asignarse memoria ésta estará limpia (todos ceros). **Los accesos de ambas tareas de una pareja en ésta región deben siempre observar la misma memoria física.**
- Sólo la tarea que **crea** una pareja puede **escribir** en los 4MB a partir de `0xC0C00000`.
- Cuando una tarea abandona su pareja pierde acceso a los 4MB a partir de `0xC0C00000`.

Para poder soportar el mecanismo de _parejas_, el sistema debe implementar las tres syscalls que serán detalladas a continuación.

### `crear_pareja()`
Para crear una pareja una tarea deberá invocar `crear_pareja()`. Como resultado de esto se deben cumplir las siguientes condiciones:
- Si **ya pertenece a una pareja** el sistema ignora la solicitud y retorna de inmediato.
- Sino, la tarea no volverá a ser ejecutada hasta que otra tarea se una a la pareja.
- La tarea que crea la pareja será llamada la **líder** de la misma y será la única que puede escribir en el área de memoria especificado.

La firma de la syscall es la siguiente:
```C
void crear_pareja();
```

### `juntarse_con(id_tarea)`
Para unirse a una pareja una tarea deberá invocar `juntarse_con(<líder-de-la-pareja>)`. Aquí se deben considerar las siguientes situaciones:
- Si **ya pertenece a una pareja** el sistema ignora la solicitud y retorna `1` de inmediato.
- Si la tarea identificada por `id_tarea` no estaba **creando** una pareja entonces el sistema ignora la solicitud y retorna `1` de inmediato.
- Si no, se conforma una pareja en la que `id_tarea` es **líder**. Luego de conformarse la pareja, esta llamada retorna `0`.

La firma de la syscall es la siguiente:
```C
int juntarse_con(int id_tarea);
```

Notar que una vez **creada** la pareja se debe garantizar que ambas tareas tengan acceso a los 4MB a partir de `0xC0C00000` a medida que lo requieran.
Sólo la líder podrá escribir y **ambas podrán leer**.

### `abandonar_pareja()`
Para abandonar una pareja una tarea deberá invocar `abandonar_pareja()`, hay tres casos posibles a los que el sistema debe responder:
- Si la tarea no pertenece a ninguna pareja: No pasa nada.
- Si la tarea pertenece a una pareja y **no es su líder**: La tarea abandona la pareja, por lo que pierde acceso al área de memoria especificada.
- Si la tarea pertenece a una pareja y **es su líder**: La tarea queda bloqueada hasta que la otra parte de la pareja abandone. Luego pierde acceso a al área de memoria especificada.

La firma de la syscall es la siguiente:
```C
void abandonar_pareja();
```

### Ejercicios
#### Parte 1: implementación de las syscalls (80 pts)
1. **(50 pts)** Definir el mecanismo por el cual las syscall `crear_pareja`, `juntarse_con` y `abandonar_pareja` recibirán sus parámetros y retornarán sus resultados según corresponda. Dar una implementación para cada una de las syscalls. Explicitar las modificaciones al kernel que sea necesario realizar, como pueden ser estructuras o funciones auxiliares.

Se puede asumir como implementadas las siguientes funciones, que se resolverán en puntos siguientes. Su uso es opcional pero recomendado:
- `task_id pareja_de_actual()`: si la tarea actual está en pareja devuelve el `task_id` de su pareja, o devuelve 0 si la tarea actual no está en pareja.
- `bool es_lider(task_id tarea)`: indica si la tarea pasada por parámetro es lider o no.
- `bool aceptando_pareja(task_id tarea)`: si la tarea pasada por parámetro está en un estado que le permita formar pareja devuelve 1, si no devuelve 0.
- `void conformar_pareja(task_id tarea)`: informa al sistema que la tarea actual y la pasada por parámetro deben ser emparejadas. Al pasar `0` por parámetro, se indica al sistema que la tarea actual está disponible para ser emparejada.
- `void romper_pareja()`: indica al sistema que la tarea actual ya no pertenece a su pareja actual. Si la tarea actual no estaba en pareja, no tiene efecto.

Estas funciones únicamente interactuan con lo que sería el "sistema de asignación de parejas"; **no tienen efecto sobre el resto del comportamiento** que deben implementar las syscalls (manejo de memoria compartida, desalojo de tarea en ejecución, etc).

A modo de ayuda el siguiente es un diagrama de estados en el que se puede ver todas las posibles circunstancias en las que una tarea puede encontrarse. Se omiten las transiciones en las que no se hace nada (una tarea sin pareja intenta abandonar a su pareja ó una tarea con pareja intenta crear otra pareja).
![Diagrama de estados descrito por el enunciado anterior](img/state-machine.svg)

2. **(20 pts)** Implementar `conformar_pareja(task_id tarea)` y `romper_pareja()`
3. **(10 pts)** Implementar `pareja_de_actual()`, `es_lider(task_id tarea)` y `aceptando_pareja(task_id tarea)`

#### Parte 2: monitoreo de la memoria de las parejas (20 pts)

4. Escribir la función `uso_de_memoria_de_las_parejas`, la cual permite averiguar cuánta memoria está siendo utilizada por el sistema de parejas.
Ésta función debe ser implementada por medio de código ó pseudocódigo que corra en nivel 0.

La firma esperada de la función es la siguiente:
```c
uint32_t uso_de_memoria_de_las_parejas();
```

Tengan en consideración los siguientes puntos:
- Cuando `abandonar_pareja` deja a una pareja sin participantes los recursos asociados cuentan como liberados.
- Una pareja sólo usa la memoria a la que intentó acceder, no se contabiliza toda la memoria a la que podría acceder en el futuro.
- Una página _es usada por el sistema de parejas_ si es accesible dentro de la región compartida para parejas por alguna de las tareas que actualmente se encuentran en pareja (o son líderes solitarios de una pareja que aún no se terminó de disolver).
- En el caso en que ambas tareas de una pareja utilizan, por ejemplo, el primer megabyte del área compartida de su pareja, la función debe retornar que hay **un megabyte** de memoria en uso por el sistema de parejas. Es decir, no se debe contabilizar dos veces el mismo espacio de memoria si ambas tareas de la pareja lo están utilizando.

## A tener en cuenta para la entrega
- Se parte de un sistema igual al del TP-SP terminado, sin ninguna funcionalidad adicional.
- Indicar todas las estructuras de sistema que deben ser modificadas para implementar las soluciones.
- Está permitido utilizar las funciones desarrolladas en las guías y TPs, siempre y cuando no utilicen recursos con los que el sistema no cuenta.
- Es necesario que se incluya una **explicación con sus palabras** de la idea general de las soluciones.
- Es necesario explicitar todas las asunciones que hagan sobre el sistema.
- Es necesaria la entrega de **código que implemente las soluciones**.
- `zero_page()` modifica páginas basándose en una **dirección virtual** (no física!)
