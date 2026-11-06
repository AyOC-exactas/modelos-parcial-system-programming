# Resolución segundo parcial 2c2025
## Ejercicio 1 - implementar solicitar y recurso_listo
Para entender todas las modificaciones necesarias para implementar solicitar y recurso_listo, pensemos en el punta a punta de la producción de un monitor:

1. Comienza la producción de un monitor mediante la int 41 (no entramos en detalle en este mecanismo para este ejercicio)
    - De acuerdo al enunciado: las direcciones 0xAAAA000 y 0xBBBB000 pueden no comenzar mappeadas.
2. Dicha producción requiere un recurso (digamos un chip), por lo que se llama `solicitar(chip)` (siendo chip de tipo recurso_t). Para que se pueda ejecutar esta acción debemos definir la syscall solicitar e implementar su rutina de atención de interrupción. Entro en detalle sobre esto en el subtitulo ["La syscall solicitar"](#la-syscall-solicitar)

Con la ejecución de `solicitar` la tarea productora de monitor queda pausada hasta que se produzca el chip. Debemos acordarnos de agregar la tarea productora de vuelta al scheduler durante la ejecución de `recurso_listo`. Pasamos a la producción del chip.

La syscall `solicitar` va a, eventualmente, disparar la ejecución de la tarea productora de chips. Esta tarea va a realizar su ejecución y eventualmente, al terminar de producir al chip:

3. La tarea productora de chips escribe a 0xAAAA000 la información del chip producido
    - La dirección 0xAAAA000 comienza desmappeada, así que saltará un Page Fault. Debemos ver como resolverlo -> Lo resuelvo en el subtítulo ["Copiando datos entre tareas"](#copiando-datos-entre-tareas)
4. Llama a la syscall `recurso_listo`. No se especifica la aridad de esta syscall, asumo que no toma parámetros ya que quiero que sea el sistema quien tenga conocimiento de para qué tarea se está realizando la producción, no la tarea.
5. La syscall recurso_listo debe:
    a. Copiar la info de 0xAAAA000 de la tarea actual a 0xBBBB000 de la tarea solicitante del recurso. -> Esto lo resuelvo en el subtítulo ["Copiando datos entre tareas"](#copiando-datos-entre-tareas)
    b. Volver a agregar a la tarea solicitante del recurso al scheduler
    c. Restaurar la tarea actual a su estado inicial con restaurar_tarea
    d. Desalojar la tarea actual (haciendo un jump far)

Los detalles de como disponibilizar la syscall recurso_listo los doy en el subtítulo ["La syscall recurso_listo"](#la-syscall-recurso_listo)

Hecho todo esto, en alguno de los próximos ticks de clock volverá a ejecutar la tarea productora de monitor, que terminará su ejecución (si tiene alguna otra dependencia, la solicitará mediante el mismo mecanismo descripto) y se desalojará. Es probable que luego de retomar su ejecución la tarea intente acceder a la dirección 0xBBBB000, pero cuando hagamos la copia de datos nos aseguraremos de dejar esta dirección mappeada, por lo que podrá acceder sin problemas.

### La syscall `solicitar`

Para disponibilizar la syscall solicitar basta con utilizar las macros y funciones auxiliares del TP-SP para hacerlo. Declaremosla como la int 90, con DPL 3 para que pueda ser llamada por tareas con nivel de privilegio de usuario.

```ASM
IDENTRY3(90)
```

Para definir la rutina de atención de la syscall, necesitamos especificar como se pasará el parámetro de tipo recurso_t. Digamos que este parámetro debe ser colocado en EAX antes de realizar el llamado a la interrupción. Entonces, la siguiente es una implementación posible de la rutina de atención:

```asm
_isr90:
    pushad
    push eax       ;pasamos el parámetro de tipo recurso_t a la función auxiliar
    call solicitar
    popad
    iret
```

En cuanto a la implementación de la función auxiliar en C, veamos a grandes rasgos el comportamiento y luego el detalle de implementación:

- Verificar si es posible producir el recurso. Si no, el comportamiento está indefinido - elijo que la tarea sea desalojada ya que quedará en un estado bloqueado infinito, lo cual es funcionalmente equivalente a desalojar la tarea productora. Para esto podemos utilizar la función auxiliar `hay_tarea_para_recurso` y en el caso de que no, `restaurar_tarea` + cambios en el scheduler.
- Si existe tarea que produzca el recurso, revisemos que dicha tarea no esté ya ocupada. Si está ocupada, debemos encontrar algún modo de esperar a que termine la producción actual o encolarnos para que nuestra producción comience al final de la anterior.
    - La función auxiliar hay_tarea_para_recurso que devuelve task solo si hay task disponible, eso nos resuelve el revisar si la tarea está ocupada o no
- Una vez que la tarea se encuentre desocupada (por enunciado esto implica que fue restaurada y desalojada del scheduler) debemos agregar a la tarea productora de chips al scheduler e informar de algún modo al sistema que el chip que se comenzará a producir ahora será producido para nuestra tarea de monitores.
- Hasta que termine la producción del chip, nuestra tarea productora de monitores debe quedar pausada. Es decir, podemos quitarla del scheduler y que vuelva a ser agregada cuando la tarea productora de chip llame a recurso_listo.

Pasemos a la implementación de la función auxiliar en C:

```c
// Aunque parte del mecanismo para informar_necesidad_recurso se debe definir con mayor detalle en el ejercicio 4, voy armando los lineamientos para terminar de explicar la parte relevante a este ejercicio
// Declaremos la siguiente variable global en sched.c:

produccion_t producciones_en_curso[MAX_TASKS] = {0}; //Digamos que se inicializa toda la memoria relevante en 0s y luego un paso de inicialización de sistema completa los recursos correspondientes a cada tarea según corresponda.

struct produccion {
    recurso_t recurso;
    task_id solicitante_actual;
}

//Tomo como dada current_task_id, que es la variable global declarada en sched.c que indica el task_id de la tarea actualmente en ejecución (no me acuerdo si este era el nombre exacto de la variable).
task_id current_task_id;

void solicitar(recurso_t recurso) {
    task_id productora_dependencia = hay_tarea_para_recurso(recurso);
    if (productora_dependencia != 0) {
        producciones_en_curso[productora_dependencia].solicitante_actual = current_task_id;
        sched_enable_task(productora_dependencia);
    } else {
        //No hay productora aún
        informar_recurso_pedido(current_task, recurso); //Implemento esta función en el ejercicio 4
    }


    //Desalojo la tarea actual y la deshabilito hasta que esté producido el recurso
    sched_disable_task(current_task_id);    
    jump_far(); 
    //la syscall recurso_listo se ocupará de volver a incluirnos en el scheduler cuando esté listo en el recurso; por lo pronto saltamos.
}
```

Donde jump_far es una función auxiliar declarada en asm que realiza el mismo proceso de llamado a jump far a sched_next_task que se utiliza en la isr32.

```asm
no lo copio pero el jump far tipico
```

### La syscall `recurso_listo`

Para disponibilizar la syscall `recurso_listo` basta con utilizar las macros y funciones auxiliares del TP-SP para hacerlo. Declaremosla como la int 91, con DPL 3 para que pueda ser llamada por tareas con nivel de privilegio de usuario.

```asm
IDENTRY3(91)
```

Como mencioné antes, esta syscall no tomará parámetros. La siguiente es la implementación en ASM de la rutina de atención de interrupción de esta syscall:

```asm
_isr91:
    pushad
    call recurso_listo
    popad
    iret
```

En cuanto a la implementación de la función auxiliar en C, recordemos el comportamiento delineado arriba:
- La syscall recurso_listo debe:
    a. Copiar la info de 0xAAAA000 de la tarea actual a 0xBBBB000 de la tarea solicitante del recurso. -> Esto lo resuelvo en el subtítulo "Copiando datos entre tareas"
    b. Volver a agregar a la tarea solicitante del recurso al scheduler
    c. Restaurar la tarea actual a su estado inicial con restaurar_tarea
    d. Desalojar la tarea actual (haciendo un jump far)

Veamos la implementación de la función auxiliar:
```c
// Recordemos que declaramos anteriormente

produccion_t producciones_en_curso[MAX_TASKS] = {0}; //Digamos que se inicializa toda la memoria relevante en 0s

struct produccion {
    recurso_t recurso;
    task_id solicitante_actual;
}

void recurso_listo() {
    task_id tarea_solicitante = para_quien_produce(current_task);
    copy_resource_info(tarea_solicitante); //La implementación de esta función la vemos más adelante
    sched_enable_task(tarea_solicitante);
    restaurar_tarea(current_task_id); 
    
    task_id prox_tarea = hay_consumidora_esperando(producciones_en_curso[current_task_id].recurso);
    if (task_id != 0) {
        producciones_en_curso[current_task_id].solicitante_actual = prox_tarea;
        // En este caso no deshabilito la tarea
    } else {
        producciones_en_curso[current_task_id].solicitante_actual = NULL;
        // ==== MODIFICACION EJ4
        producciones_en_curso[tarea_solicitante].recurso_solicitado = NULL;
        // ==== FIN MODIFICACION EJ4
        sched_disable_task(current_task_id);
        jump_far(); //Ver implementación en subtítulo anterior
    }
}
```

Cabe mencionar que probablemente en algún momento en esta función deberíamos informar que esta tarea está nuevamente disponible para producir más chips. Queda para el ejercicio 4 modificar esta función para definir ese mecanismo, ya que en este punto se da la función `hay_tarea_para_recurso` como resuelta.

### Copiando datos entre tareas
Para copiar los datos de producción entre tareas debemos:
- Resolver el page fault cuando se accede a 0xAAAA000 o 0xBBBB000 y no están mappeadas
- Resolver la auxiliar `copy_resource_info` que:
    - Busca la dirección física mappeada a la virtual 0xBBBB0000 en el esquema indicado por el CR3 asociado a la tarea solicitante
        - Si el mappeo no está hecho, se lo dejo hecho a la tarea solicitante, asignando una página física nueva del área libre de tareas.
        - Si está hecho, tomo esa física
    - Mappea esa física a una virtual temporal en mi CR3 actual, digamos 0xBABAB000
    - Copia toda la página 0xAAAA000 a 0xBABAB000
    - Deshace el mappeo temporal a 0xBABAB000

En pos de resolver esos puntos, modifico la función `page_fault_handler` del TPSP e implemento la auxiliar descripta.

```c
bool page_fault_handler(vaddr_t virt) {
    print("Atendiendo page fault...", 0, 0, C_FG_WHITE | C_BG_BLACK);
    // En el TPSP original acá teníamos el chequeo de acceso a sección on demand, pero en este parcial no hay mención de que se use ese mecanismo, así que no lo incluyo.

    virt_frame = (virt >> 12) << 12
    if (virt_frame != 0xAAAA000 && virt_frame != 0xBBBB000) {
        //No estamos dentro de las direcciones de page fault esperado. Salimos.
        return false;
    }
    // Estamos intentando acceder a alguna de las páginas de info de producción. Tenemos que inicializar esa memoria
    mmu_map_page(rcr3(), virt & ~(0xFFF), mmu_next_free_user_page(),
                MMU_U | MMU_W | MMU_P);
    return true;
}

void copy_resource_info(task_id tarea_solicitante) {
    // - Busca la dirección física mappeada a la virtual 0xBBBB0000 en el esquema indicado por el CR3 asociado a la tarea solicitante
    // Es una modificación de copy_page, se podía también buscar las físicas correctas y llamar copy_page directo.
    cr3_sol = get_cr3(tarea_solicitante); //Esto va a buscar al array de tss el cr3 del task_id dado
    pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3_sol);

    int new_phys = 0;
    int pdi = VIRT_PAGE_DIR(0xBBBB000);
    if ((pd[pdi].attrs & MMU_P)) {
        // - Si el mappeo está hecho, tomo esa física
        pt_entry_t* pt = (pt_entry_t*)MMU_ENTRY_PADDR(pd[pdi].pt);
        int pti = VIRT_PAGE_TABLE(0xBBBB000);
        if ((pt[pti].attrs & MMU_P))
            new_phys = MMU_ENTRY_PADDR(pt[pti].page);
    }

    if (new_phys == 0) {
        //- Si el mappeo no está hecho, se lo dejo hecho a la tarea solicitante, asignando una página física nueva del área libre de tareas.
        new_phys = mmu_next_free_user_page();
        mmu_map_page(cr3_sol, 0xBBBB000, new_phys, MMU_P | MMU_U | MMU_W);
    }

    //- Mappea esa física a una virtual temporal en mi CR3 actual, digamos 0xBABAB000
    mmu_map_page(rcr3(), 0xBABAB000, new_phys, MMU_P | MMU_W);
    //- Copia toda la página 0xAAAA000 a 0xBABAB000
    ciclo_copia_4k(rcr3(), 0xAAAA000, 0xBABAB000); //Es un for loop que hace ptr2[i] = ptr1[i]

    //- Deshace el mappeo temporal a 0xBABAB000
    mmu_unmap_page(rcr3(), 0xBABAB000);
}
```

## Ejercicio 2 - implementar arranque manual -> cambiar este ej a 20 puntos y el 3 a 25?
Dado que el arranque manual se inicializa mediante una interrupción externa de hardware, pasará por el PIC. Partamos de un PIC configurado del mismo modo que hacemos para el TPSP.

Declaremos la int 41, con DPL 0 ya que es llamada por el PIC -> privilegio de kernel.

```asm
IDENTRY0(41)
```

La rutina de atención de la interrupción:
```asm
_isr41:
    pushad
    picfinish ;importante!

    call obtener_recurso_arranque_manual ;devuelve el recurso en eax
    call solicitar ;definimos anteriormente que el parámetro se pasa por eax

    .fin:
    popad
    iret
```

```c
recurso_t obtener_recurso_arranque_manual() {
    // La dirección 0xFAFAFA está comprendida dentro del identity mapping del kernel, por lo que puedo acceder directamente
    return *(recurso_t*)0xFAFAFA
}
```

## Ejercicio 3 - implementar restaurar_tarea
Para restaurar una tarea a su estado inicial debemos:
- Restaurar los valores de EIP, ESP y EBP, que son los importantes que se modifican durante la ejecución de la tarea (los regs de propósito general también, pero creo que es razonable asumir que la tarea no espere que sus regs de propósito general tengan ninguna inicialización particular para funcionar correctamente).
- Desmappear toda la memoria mappeada ("reiniciar" su esquema de paginación)
  - De acuerdo al enunciado basta con desmappear 0x0BBBB000, creo que también habría que desmappear 0x0AAAA000 por si se usó
- Reiniciar las estructuras del sistema relacionadas a solicitar y recurso_listo

Vamos a hacerlo:
- Tenemos que modificar los valores de los registros mencionados en la pila lvl 0 así cuando se haga el popad antes del iret ya quedan bien. Para encontrar estos valores podemos aprovechar que en la TSS siempre tenemos el esp0 inicial, y que sabemos exactamente qué cosas se apilan desde el llamado a recurso_listo que es el único lugar donde se llama restaurar_tarea.
- Vamos a las page table entries correspondientes a 0xBBBB000 y 0xAAAA000 y ponemos el bit de presente en 0. Basta con que hagamos eso, no es necesario limpiar los demás atributos.
- En cuanto a estructuras del sistema, basta con colocar producciones_en_curso[current_task_id].solicitante_actual en 0. Esto me parece más correcto que lo haga recurso_listo.

Hecho todo esto, se efectivizará la limpieza de registros la próxima vez que el clock asigne a esta tarea para ejecución. Quitar esta tarea del scheduler y realizar el jmp far son responsabilidades que quedan por fuera de esta función (se incluyeron en recurso_listo).

Pasamos a implementarla:

```c
void restaurar_tarea() {
    tss_t* tss_actual = &tass_tasks[current_task_id];
    /* Versión alternativa: obtener tss de la gdt
    desc_tss = gdt[sched_tasks[current_task_id].selector >> 3];
    // Digamos que tenemos una macro GDT_BASE que reconstruye la base de un descriptor de segmento a partir de las tres partes separadas
    tss_t* tss_actual = (tss_t*) GDT_BASE(desc_tss);
    */
    uint32_t* pila = tss_actual.esp0;
    pila[OFFSET_EIP_PUSHAD] = VALOR_INICIAL_EIP;
    pila[OFFSET_ESP_PUSHAD] = VALOR_INICIAL_ESP;
    pila[OFFSET_EBP_PUSHAD] = VALOR_INICIAL_EBP;

    mmu_unmap_page(rcr3(), 0xAAAA000);
    mmu_unmap_page(rcr3(), 0xBBBB000);
}
```

## Ejercicio 4 - implementar hay_tarea_para_recurso y hay_consumidora_esperando
En cuanto a las preguntas:
- ¿Cómo modificarían el sistema para almacenar qué recurso produce cada tarea?
  - Podemos agregar el array `produccion_t producciones_en_curso[MAX_TASKS] = {0};` que mostré antes
- ¿Cómo modificarían el sistema para llevar registro de qué tareas están esperando a que se liberen productoras de su recurso solicitado? 
  - Podríamos agregar un tipo extra al enum task_state_t que sea "TASK_WAITING_FOR_RESOURCE" y un nuevo atributo `recurso_solicitado` a `produccion_t`, de modo que el array producciones_en_curso indique para cada tarea qué recurso se está produciendo, para quién y qué recurso está esperando. El struct resulta:

```c
struct produccion {
    recurso_t recurso_producido; //Cambio el nombre por claridad. Campo no nulleable
    task_id solicitante_actual; //Nullable
    recurso_t recurso_solicitado; //Nullable
}
```

Pasemos a la implementación de las funciones:
```c
task_id hay_tarea_para_recurso(recurso_t recurso) {
    for (int i=MAX_TASKS-1; i > 0; i--) {
        if (producciones_en_curso[i].recurso_producido == recurso && producciones_en_curso[i].solicitante_actual == NULL)
            break; //Hay alguien que produce el recurso que no está ocupado actualmente
        }
    //El task_id 0 corresponde a la task idle, que no es productora, por lo que corto mi ciclo antes
    //Si se llegó a i=0, no se encontró productora para el recurso
    return i;
}
```

```c
task_id hay_consumidora_esperando(recurso_t recurso) {
    for (int i=MAX_TASKS-1; i > 0; i--) {
        if (producciones_en_curso[i].recurso_solicitado == recurso)
            break;
        }
    //El task_id 0 corresponde a la task idle, que no es productora, por lo que corto mi ciclo antes
    //Si se llegó a i=0, no se encontró productora para el recurso
    return i;
}
```

```c
void informar_recurso_pedido(task_id task, recurso_t recurso) {
    producciones_en_curso[task].recurso_solicitado = recurso;
}
```

```c
task_id para_quien_produce(task_id task) {
    return producciones_en_curso[task_id].solicitante_actual;
}
```

Más arriba había dejado el comentario:
> Cabe mencionar que probablemente en algún momento en esta función deberíamos informar que esta tarea está nuevamente disponible para producir más chips. Queda para el ejercicio 4 modificar esta función para definir ese mecanismo, ya que en este punto se da la función `hay_tarea_para_recurso` como resuelta.

En ese mismo ejercicio ya estaba seteando `producciones_en_curso[i].solicitante_actual` en NULL al terminar la producción así que basta con eso para que el sistema sepa que se terminó la producción (así identificamos si hay producción en curso en `hay_tarea_para_recurso`). Lo que falta es cuando se termina la producción, indicar que la tarea que pidió ya no tiene el pedido pendiente. Bastaría con agregar estas lineas antes del jump far:

```c
producciones_en_curso[tarea_solicitante].recurso_solicitado = NULL;
```