  Nos encargaron el desarrollo de un kernel para la nueva consola Orga Génesis que cuenta con un procesador x86. Para ahorrar trabajo vamos a tomar nuestro kernel y expandirlo para permitir la ejecución de juegos mediante cartuchos, que contendrán el código y los recursos gráficos (sprites de personajes, fondos). Se incorpora entonces un lector de cartuchos que, entre otras cosas, copiará los gráficos del cartucho a un buffer de video de 4 kB en memoria. Cada vez que el buffer esté lleno y listo para su procesamiento, el lector se lo informará al kernel mediante una interrupción externa mapeada al IRQ 40 de nuestro x86.
  
  Distintas tareas se ocuparán de mostrar los gráficos en pantalla y de realizar otros pre/postprocesamientos de imagen on-the-fly. Para esto, el sistema debe soportar que las tareas puedan acceder al buffer de video a través de los siguientes mecanismos:

	- DMA (Direct Memory Access): se mappea la dirección virtual `0xBABAB000` del espacio de direcciones de la tarea directamente al buffer de video.

	- Por copia: se realiza una copia del buffer en una página física específica y se mapea en una dirección virtual provista por la tarea. Cada tarea debe tener una copia única.

Dicho buffer se encuentra en la dirección física de memoria `0xF151C000` y solo debe ser modificable por el lector de cartuchos.

 Para solicitar acceso al buffer, las tareas deberán informar que desean acceder a él mediante una syscall `opendevice` habiendo configurado el tipo de acceso al buffer en la dirección virtual `0xACCE50` (mapeada como r/w para la tarea). Allí almacenará una variable  `uint8_t acceso` con posibles valores `0`, `1` y `2`. El valor `0` indica que la tarea no accederá al buffer de video, `1` que accederá mediante DMA y `2` que accederá por copia. De acceder por copia, la dirección virtual donde realizar la copia estará dada por el valor del registro `ECX` al momento de llamar a `opendevice`.

 El sistema no debe retomar la ejecución de estas tareas hasta que se detecte que el buffer está listo y se haya realizado el mapeo DMA o la copia correspondiente. Una vez que la tarea termine de utilizar el buffer, deberá indicarlo mediante la syscall `closedevice`. En ésta se debe retirar el acceso al buffer por DMA o dejar de actualizar la copia, según corresponda.

 La interrupción de buffer completo será la encargada de dar el acceso correspondiente a las tareas que lo hayan solicitado y actualizar las copia del buffer "vivas". Es deseable que cada tarea que accede por copia mantenga una única copia del buffer para no ocupar la memoria innecesariamente.

## Ejercicio 1:
- a) Programar la rutina que atenderá la interrupción que el lector de cartuchos generará al terminar de llenar el buffer.
	- Consejo: programar una función deviceready y llamarla desde esta rutina.
- b) Programar las syscalls opendevice y closedevice.
- Cuentan con las siguientes funciones ya implementadas:
	- void buffer_dma(pd_entry_t* pd) que dado el page directory de una tarea realice el mapeo del buffer en modo DMA.
	- void buffer_copy(pd_entry_t* pd, paddr_t phys, vaddr_t virt) que dado el page directory de una tarea realice la copia del buffer a la dirección física pasada por parámetro y realice el mapeo a la dirección virtual pasada por parámetro.
## Ejercicio 2:
- a) Programar la función void buffer_dma(pd_entry_t* pd)
- b) Programar la función void buffer_copy(pd_entry_t* pd, paddr_t phys, vaddr_t virt)
