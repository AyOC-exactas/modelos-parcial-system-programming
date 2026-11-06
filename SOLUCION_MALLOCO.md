## Primera parte

#### Definiendo Malloco:

Nos piden crear una syscall para que cada tarea pueda pedir memoria de una manera dinámica. Como definimos nosotros al desarrollar el TP, cada tarea cuenta con una estructura de paginación de 4MB de tamaño. Por esto, nos restringen que el pedido total que puede hacer una tarea sea de también 4MB. Para esto, tendré que ir guardando los pedidos que una tarea hace de memoria.


#### Definiendo el array de registro de reservas:

<span style="color:yellow">Nota: se aclaró durante el parcial que el pedido de memoria de las tareas es múltiplo de 4KB.</span>

Necesito definir una estructura que almacene las reservas de memoria de las tareas del sistema. Para cada tarea, puedo guardar un array en el que cada elemento es una reserva de memoria. Como a lo sumo una tarea puede pedir hasta 4MB de memoria y cada reserva es de al menos 4KB, a lo sumo tendré 1024 reservas de memoria por cada tarea, ese será el tamaño máximo del array de reservas.

Con esto hecho empiezo a describir como va a funcionar todo:

- Una tarea pide memoria a Malloco, pueden ocurrir los siguientes casos:
    - Esa tarea aún le queda memoria restante (puede ser la primera vez que pide memoria o no), podemos asignarle por lo que agregamos un elemento al arreglo de registro de reservas (que llamaré reservas_por_tarea) con el formato de un struct: <id tarea, array de reservas, cantidad de reservas> donde una reserva es un struct <dir virtual de inicio, cantidad de paginas, si se liberó o no>. . Además modificaré el struct sched_task_entry tal que para cada tarea, además se guarde como atributo la cantidad de páginas que pidió al momento. Tal que pueda ir chequeando que no se exceda de la memoria asignable permitida.
    - Esa tarea no tiene memoria restante, osea la cantidad de páginas (en total) que pidió esa tarea alcanzó ya el número 1024. Entonces devuelvo NULL.
- Una tarea efectivamente quiere acceder a la memoria que reservó, cuando intente hacerlo caerá la exepción *Page Fault*. Esto se debe a que intentó acceder a un lugar de memoria que aún concretamente no le pertenece. Por eso, basta con modificar el `page_fault_handler` que definimos en el TP de modo que además de chequear la memoria **on-demand**, ahora chequee también si la posición de memoria que quiso acceder la tarea le pertenece.


### MALLOCO:

Comienzo por agregar la syscall **Malloco** a la IDT:

```c
IDT_ENTRY3(99);
```

Ahora la defino en asm:

```
global _isr99

_isr99:
    pushad 

    ; tomo que el parámetro se pasa por edi y se lee en eax
    push edi
    call malloco
    add esp, 4
    mov [esp + offset_EAX], eax

    popad 
    iret
```


Modifico el struct sched_entry_t que mencione anteriormente:

```c
typedef struct {
  int16_t selector;
  task_state_t state;
  int16_t pages;        // cantidad de páginas pedidas (default 0)
} sched_entry_t;
```

Defino como va a ser mi arreglo de reservas de memoria:

```c
typedef struct{
    vaddr_t virt;
    int16_t cantPaginas;     
    uint8_t chau;       // 0 no, 1 si
} reserva_t;

typedef struct{
    int8_t tarea;
    reserva_t* reservas; //array de reservas
    int16_t reservas_size //cantidad de elementos en el array
} reservas_por_tarea_t;

reservas_por_tarea_t* reservas_por_tarea[MAX_TASKS];
```

```c
#define VIRT_RESERVABLE 0xA10C0000

void* malloco(size_t size){
    // uso fuertemente el id asignado a cada tarea proveniente del arreglo sched_tasks

    // verifico que la tarea que llamó a malloco tenga espacio libre para asignar
    uint16_t this_task_pages = sched_tasks[current_task].pages; 
    // 4096 bytes = 4KB = el tamaño de una página
    if((this_task_pages + (size/4096)) > 1024){ 
        return NULL;
    }

    // si estoy acá había memoria para asignar

    int16_t indice_nueva_reserva = reservas_por_tarea[current_task].reservas_size;
    vaddr_t virtual_reservada = VIRT_RESERVABLE + this_task_pages*1024; 

    reservas_por_tarea[current_task].reservas[indice_nueva_reserva].virt = virtual_reservada;
    reservas_por_tarea[current_task].reservas[indice_nueva_reserva].cantPaginas = size/4096;
    reservas_por_tarea[current_task].reservas[indice_nueva_reserva].chau = 0;

    reservas_por_tarea[current_task].reservas_size += 1; //aumento en 1 el tamaño ocupado del array
   
    sched_tasks[current_task].pages += size/4096;

    return (void*)(virtual_reservada);
}
```

---
### ACCESO A MEMORIA:

Ahora defino qué sucederá si una **tarea quiere acceder a las posiciones de memoria que reservó**:

Modifico el page_fault_handler para que chequee si una tarea quiso acceder a su memoria reservada (pero que aún no le fue asignada memoria):

```c
bool page_fault_handler(vaddr_t virt) {
    print("Atendiendo page fault...", 0, 0, C_FG_WHITE | C_BG_BLACK);
    uint32_t cr3 = rcr3();
    // Chequeemos si el acceso fue dentro del area on-demand
    if(virt >= ON_DEMAND_MEM_START_VIRTUAL && virt < ON_DEMAND_MEM_END_VIRTUAL){
        // En caso de que si, mapear la pagina
        mmu_map_page(cr3, virt, ON_DEMAND_MEM_START_PHYSICAL, (MMU_P | MMU_U | MMU_W));
        return true;
    }
    // Chequeo si el acceso fue dentro de su memoria registrada
    for(uint16_t i=0; i<1024; i++){
        if(!tiene_reserva_valida(virt)){
            sched_disable_task(current_task);
            liberar_memo_tarea();
            return false;
        }
        paddr_t phy = mmu_next_free_user_page();
        mmu_map_page(cr3, virt, phy, (MMU_P | MMU_U | MMU_W)); // tomo estos atributos porque se aclara que el acceso puede ser por lectura o escritura
        zero_page((virt >> 12) << 12); // se pasa la dirección base de la página virtual a zero_page para inicializarla en cero

        return true;
    }

  
    return false;
```




Defino la rutina de atención del page fault con estas modificaciones:

```
global _isr14

_isr14:
	; Estamos en un page fault.
	pushad

    ; Llamar rutina de atención de page fault, pasandole la dirección que se intentó acceder
    mov ecx, cr2
    push ecx
    call page_fault_handler
    pop ecx

    cmp al, 1
    jmp .fin

    .ring0_exception:
	; Si llegamos hasta aca es que cometimos un page fault fuera del area compartida o del área reservada por la tarea

    ; Continuamos con la siguiente tarea
    call sched_next_task

    mov word [sched_task_selector], ax

    jmp far [sched_task_offset]

    .fin:
	popad
	add esp, 4 ; error code
	iret
```

Defino ahora las funciones auxiliares usadas en `page_fault_handler`:

```c
// AUXILIARES
// --------------------------------------------------------
bool tiene_reserva_valida(vaddr_t virt){
    // cuando esto se ejecuta estoy en el contexto de un page fault mientras una tarea se ejecutaba, 
    // por lo que puedo acceder al contenido de esa tarea libremente
    uint16_t cantReservas = reservas_por_tarea[current_task].reservas_size;
    for(uint16_t i=0; i<cantReservas; i++){
        vaddr_t start_virt_addr = reservas_por_tarea[current_task].reservas[i].virt;
        uint16_t this_block_memo = reservas_por_tarea[current_task].reservas[i].cantPaginas;
        uint8_t is_chau = reservas_por_tarea[current_task].reservas[i].chau;

        // si la memoria a la que se quiso acceder esta en el rango de un bloque y además en un bloque no liberado
        if(virt >= start_virt_addr && virt < (start_virt_addr+(this_block_memo*PAGE_SIZE)) && !is_chau){
            return true;
        }
    }
    // si llegue acá sin haber salido de la función no se encontró ninguno (o no había memoria disponible o estaba liberada)
    return false;
}

void liberar_memo_tarea(){
    // idem funcion anterior
    uint16_t cantReservas = reservas_por_tarea[current_task].reservas_size;
    for(uint16_t i=0; i<cantReservas; i++){
        reservas_por_tarea[current_task].reservas[i].chau = 1;
    }
    return;
}
```

---
### CHAU:

Por último, implemento la syscall que libera tarea. Por lo que defino otra entrada a la IDT:

```c
IDT_ENTRY3(100);
```

Y ahora defino su rutina de atención:

```
global _isr100

_isr100:
    pushad

    push edi
    call chau
    add esp, 4

    popad 

    iret
```

y defino como va a comportarse chau. Recibirá una dirección virtual que corresponde con la dirección más baja de un bloque reservado por la tarea llamadora (previamente con malloco). Buscaré la dirección virtual en el arreglo de reservas correspondiente y lo marcaré para liberar.

```c
void chau(void* ptr){
    vaddr_t virt_a_borrar = (vaddr_t)ptr;

    uint16_t cantReservas = reservas_por_tarea[current_task].reservas_size;
    for(uint16_t i=0; i<cantReservas; i++){
        if (reservas_por_tarea[current_task].reservas[i].virt == virt_a_borrar){
            reservas_por_tarea[current_task].reservas[i].chau = 1;
            return;
        }       
    }
    return;
}
```

Lo que faltaría sería definir la tarea de nivel 0 que se encargue de realmente liberar toda la memoria que fue solicitada. Para esto, con respecto al tp, ahora tendremos una nueva entrada en la gdt para la tss de esta tarea. Además las páginas para su estructura: una para el directorio, dos para el stack (aunque podría ser una ya que al ser de nivel 0 siempre se usará la misma), una para el page table y en principio dos para el código. 

Es necesario que esta tarea sea de nivel 0 porque en otro caso, no sería capaz de acceder al arreglo de registro de reservas para liberar aquellas que lo hayan solicitado. Y particularmente, para desmapear 

```c
void garbage_collector(){
    while(true){
        for(uint16_t i=0; i<MAX_TASKS; i++){
            
            uint16_t cantReservas = reservas_por_tarea[i].reservas_size;
            uint32_t cr3 = conseguir_cr3(reservas_por_tarea[i].tarea);

            for(uint16_t j=0; j<cantReservas; j++){
                if(!reservas_por_tarea[i].reservas[j].virt){
                    continue;
                }
                if(reservas_por_tarea[i].reservas[j].chau == 0){
                    continue;
                }            
                
                // desmapeo el bloque que haya sido solicitado liberarse (osea las páginas contenidas en ese bloque)
                cantPaginas_a_liberar = reservas_por_tarea[i].reservas[j].cantPaginas;
                for(uint16_t k=0; k<cantPaginas_a_liberar; k++){
                    mmu_unmap_page(cr3, reservas_por_tarea[i].reservas[j].virt+(PAGE_SIZE*k));
                } 

                // limpio el elemento de la reserva
                reservas_por_tarea[i].reservas[j].virt = 0;
                reservas_por_tarea[i].reservas[j].cantPaginas = 0;
                reservas_por_tarea[i].reservas[j].chau = 0;
            }
            
        }
    }
    return;
}

uint32_t tick_amount(){
    uint32_t res =  (ENVIRONMENT->tick_count)%100;
    return res;
}

uint32_t conseguir_cr3(int8_t tarea){
    tss_t tss_tarea = tss_tasks[tarea];

    return tss_tarea.cr3;
}
```

Esta tarea no deberá estar en el scheduler, ya que se debe ejecutar particularmente cada 100 ticks del clock. Es por eso que en la interrupción del clock, si se llegan a los 100 ticks, basta con hacer un jmp a la tarea, que se ejecute y que ante la próxima interrupción del clock se retome el orden de round robin original.

Modifico entonces la interrupción del clock:

```c
; garbage collector (gc)
%define SELECTOR_TAREA_GC (0x11 << 3)    // la agrego una despúes que la última agregada en el TP

global _isr32

_isr32:
    pushad
    ; 1. Le decimos al PIC que vamos a atender la interrupción
    call pic_finish1
    call next_clock

    ; 2. Me fijo cuantos ticks van
    call tick_amount

    cmp eax, 0
    jnz .noCien

    .Cien:
    mov word [sched_task_selector], SELECTOR_TAREA_GC

    jmp far [sched_task_offset]

    .noCien:
    ; 3. Realizamos el cambio de tareas en caso de ser necesario
    call sched_next_task

    cmp ax, 0
    je .fin

    str bx
    cmp ax, bx
    je .fin

    mov word [sched_task_selector], ax

    jmp far [sched_task_offset]


    .fin:
    ; 3. Actualizamos las estructuras compartidas ante el tick del reloj
    call tasks_tick


    ; 4. Actualizamos la "interfaz" del sistema en pantalla
    call tasks_screen_update


    popad

    iret
```

NOTA: Muchos tipos de datos (particularmente los uint16_t) fueron elegidos para que admitan hasta 1024. Cómo puede haber un número muy grande de tareas, no necesariamente vaya a ser ese el tamaño del dato y deba ser cambiado a algo más grande.
