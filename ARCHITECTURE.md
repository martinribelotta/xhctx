# RISC-V Event-Activated Context Architecture (XCTX)

## Architecture Brief (Spanish)

> **Nota para Claude Code:** Este documento está escrito en español para
> facilitar la discusión de diseño. **Toda la documentación técnica
> generada (especificación ISA, diagramas, documentos Markdown y texto
> normativo) debe escribirse en inglés**, siguiendo el estilo de las
> especificaciones oficiales de RISC-V International.

## 1. Objetivo

Diseñar una **extensión experimental para RISC-V** que reemplace el
modelo tradicional de interrupciones (`trap → ISR → mret`) por un modelo
de **contextos de ejecución persistentes activados por eventos**.

El modelo clásico es:

``` text
Evento
  ↓
Trap
  ↓
ISR
  ↓
mret
```

El modelo propuesto es:

``` text
Evento
  ↓
Event Fabric
  ↓
Contexto persistente
  ↓
Abandono
  ↓
Siguiente contexto
```

La unidad planificable deja de ser la interrupción y pasa a ser el
**Execution Context**.

El objetivo principal es obtener una arquitectura de microcontrolador
con:

-   Latencia de activación determinista y mínima.
-   Cero salvado/restauración de registros por software.
-   Ausencia de prólogo y epílogo de ISR.
-   Eliminación del concepto de `mret`.
-   Contextos residentes completamente en hardware.
-   Event Fabric con arbitraje por prioridad.
-   Implementación viable en un RV32E pequeño.

------------------------------------------------------------------------

## 2. Principios de diseño

1.  Los contextos son persistentes y nunca se crean ni destruyen durante
    la ejecución.
2.  Los eventos activan contextos; no existen ISR efímeras.
3.  El software nunca escribe directamente el contexto activo.
4.  Todo el estado de ejecución está completamente bankeado.
5.  El cambio de contexto consiste únicamente en seleccionar otro banco.
6.  El hardware arbitra eventos; no implementa un scheduler RTOS.
7.  El comportamiento debe ser determinista y fácilmente verificable.
8.  La ISA debe permanecer compatible con RV32E siempre que sea posible.

------------------------------------------------------------------------

## 3. Modelo arquitectónico

La unidad fundamental de ejecución es el **Execution Context**.

### Relación con el modelo de hart de RISC-V

Cada contexto se comporta arquitectónicamente como un **hart** de
RISC-V: tiene su propio PC, su propio register file, su propio
`mhartid` y (Sección 8) su propio `mepc`/`mcause`. Esto permite
reutilizar directamente el modelo multi-hart existente de RISC-V
(numeración de harts, CSRs por hart, Debug Spec multi-hart, Sección
14) en vez de inventar un concepto arquitectónico nuevo.

La diferencia con un hart RISC-V estándar es deliberada: **`mret` no
existe en este modelo** (Principio 2). Un contexto nunca retorna de
una excepción localmente — el yield de control siempre pasa por
`mctxctl` (Sección 10), y una excepción síncrona no salta a un vector
`mtvec` local, sino que se entrega como un evento a otro contexto
(Sección 10, Caso D). El destino de `mret` y `wfi` como instrucciones
del ISA base se trata en la Sección 13. Todo lo demás del modelo de
hart se preserva; solo el mecanismo de trap-return y de espera cambia.

Esto define un modelo de **harts preemptibles y reanudables**: a
diferencia de harts RISC-V estándar (que se asumen independientes,
cada uno con su propio recurso de ejecución), los `N` contextos de
XCTX comparten un único pipeline físico y son desalojados/reanudados
por hardware (Sección 7), no por software ni por independencia física.

Implementación de referencia:

-   RV32E
-   Pipeline de 3 etapas
-   16 contextos hardware
-   `ctx_id` de 4 bits

Cada contexto contiene:

-   PC
-   Registros x0--x15
-   IF/ID
-   ID/EX
-   EX/WB
-   Estado de control del pipeline
-   Estado arquitectónico específico del contexto

Todo este estado está completamente bankeado.

El procesador únicamente mantiene internamente:

``` text
active_ctx : 4 bits
```

Cambiar de contexto significa conceptualmente:

``` text
active_ctx ← new_ctx
```

No existe copia de registros ni reconstrucción del pipeline.

### Tagging de contexto en el pipeline

Solo la etapa WB escribe el register file, y las lecturas de registros
se direccionan directamente por `{ctx_id, reg_num}`, así que nunca hay
conflicto de puerto de lectura ni de escritura entre contextos. Sin
embargo, el `ctx_id` igual tiene que viajar junto con la instrucción a
través de cada registro de pipeline (IF/ID, ID/EX, EX/WB), porque
`active_ctx` puede ya apuntar al nuevo contexto para cuando una
instrucción más vieja llega a retirarse en WB. Sin un tag por etapa,
WB podría escribir en el banco de un contexto equivocado.

Un cambio de contexto (voluntario vía `mctxctl`, Sección 10, o forzado
por arbitraje, Sección 7) se implementa como un squash-and-redirect
estándar, de costo idéntico al de un salto resuelto en un ciclo:

-   La instrucción que ya fue fetcheada detrás de la instrucción que
    dispara el cambio (mismo contexto, PC secuencial) se descarta.
-   El PC bankeado del contexto saliente queda congelado en el punto
    de reanudación correcto sin necesidad de que esa instrucción
    descartada llegue a ejecutarse.
-   El fetch del contexto entrante arranca en el ciclo inmediato
    siguiente.

Esto requiere que `mctxctl` (y cualquier condición de cambio forzado
por hardware) resuelva dentro de una sola etapa de pipeline, igual que
un salto. Bajo esa restricción, la activación de contexto es **de 1
ciclo, sin burbuja** — no un drenado basado en stall.

------------------------------------------------------------------------

## 4. Contextos completamente bankeados

El estado completo del procesador pertenece al contexto.

Cada contexto dispone de su propio banco de:

-   Program Counter
-   Register File
-   Pipeline Registers
-   Pipeline Control State

La implementación debe priorizar la simplicidad arquitectónica antes que
optimizaciones prematuras de área.

------------------------------------------------------------------------

## 5. Event Fabric

Los eventos pueden originarse desde:

-   IRQ externas
-   Timers
-   DMA
-   GPIO
-   Ethernet
-   Periféricos
-   Aceleradores
-   Software

Un evento contiene:

``` text
event_id
event_data
```

Ambos campos tienen ancho `XLEN` (32 bits en RV32E), simplificando la
entrada de FIFO a un par de registros de ancho uniforme y evitando
empaquetado de bits.

Si múltiples ocurrencias físicas de una misma fuente se combinan en un
único evento lógico antes de llegar al Event Fabric (coalescing) es una
decisión del **event dispatcher** de la fuente (ej. un periférico o su
controlador), no de este mecanismo. El Event Fabric y las FIFOs de la
Sección 6 tratan cada `event_id`+`event_data` que reciben como una
unidad atómica y opaca — no coalescen ni dividen nada por sí mismos.

El Event Router implementa:

``` text
event_id → context_id
```

La tabla de ruteo es configurable por firmware privilegiado.

Todo evento, sea de origen hardware o software, se rutea de forma
uniforme vía `event_id → context_id` — no hay excepción a esta regla.
Un rango reservado de ruteo identidad, `SW_TARGET_BASE + ctx_id`, le
permite al software direccionar un contexto destino arbitrario (ver
Sección 9) sin romper esa uniformidad.

------------------------------------------------------------------------

## 6. FIFOs por contexto

Existe una FIFO independiente para cada contexto.

``` text
FIFO[0]
FIFO[1]
...
FIFO[15]
```

Cada entrada almacena únicamente:

-   event_id
-   event_data

El contexto destino es implícito por pertenecer a esa FIFO.

### Bypass obligatorio

Los eventos **no deben encolarse por defecto**.

Si un evento puede ser servido inmediatamente, debe activar directamente
el contexto.

La FIFO solo se utiliza cuando el evento no puede despacharse en el
ciclo siguiente.

Este requisito es fundamental para lograr latencia mínima.

### Eventos simultáneos

Dos eventos que llegan en el mismo ciclo con destinos distintos no
compiten entre sí: cada uno se empuja a su propia FIFO
independientemente. Si ambos llegan con el mismo destino, se encolan
normalmente (orden de llegada dentro de esa FIFO). El único caso que
requiere desempate es cuando más de un evento es simultáneamente
elegible para bypass (Sección 7): el desempate usa la misma prioridad
fija por `context_id` ya definida — el de mayor prioridad se despacha
este ciclo, el otro cae a su FIFO. No hace falta hardware de arbitraje
adicional más allá del Priority Encoder que ya existe.

Si el orden relativo de entrega entre un evento servido por bypass y
uno que quedó encolado se preserva o no es **definido por
implementación** — la arquitectura no lo garantiza en ningún sentido.

### Política de overflow

La profundidad de la FIFO es definida por implementación (1, 2, 4, 16,
128... entradas, según los recursos disponibles). Sin importar la
profundidad, el overflow se maneja con **backpressure al productor**,
no con descarte silencioso — esto preserva el objetivo de determinismo
ante fuentes de eventos en ráfaga.

------------------------------------------------------------------------

## 7. Prioridad

Primera versión:

``` text
prioridad = context_id
```

Por lo tanto:

``` text
C15 > C14 > ... > C1 > C0
```

El siguiente contexto se obtiene mediante:

-   señal `!empty` de cada FIFO
-   Priority Encoder
-   generación de `next_ctx`

No debe existir un scheduler programable.

### Modelo totalmente preemptivo

El modelo **no es cooperativo**. La prioridad fija por `context_id` se
evalúa de forma continua, no solo en el abandono voluntario:

-   Cuando el contexto destino de un evento entrante tiene prioridad
    **mayor** que `active_ctx`, desaloja de inmediato, usando el mismo
    camino de squash-and-redirect sin burbuja de la Sección 3.
-   Cuando tiene prioridad **igual o menor**, se encola (las reglas de
    FIFO/bypass de la Sección 6 aplican sin cambios).
-   `mctxctl` (abandono voluntario, Sección 10) sigue disponible como
    un disparador separado del mismo camino de redirección.

Esta preemption por prioridad puede pausarse temporalmente con
`mstatus.MIE = 0` (Sección 8), para proteger secuencias
read-modify-write cortas contra recursos externos compartidos.

Un evento originado por timer/watchdog **es la única excepción** a esa
pausa: no respeta `MIE`, exactamente igual que una NMI en RISC-V
estándar — de lo contrario, un contexto con `MIE = 0` permanente
reintroduciría el livelock que este mecanismo existe para prevenir.
Fuera de esa particularidad, el evento de timer/watchdog no usa un
camino de hardware distinto: es un evento normal cuyo destino es el
contexto actualmente activo, ruteado como cualquier otro y sujeto a la
misma regla de prioridad.

Esto es, en los hechos, un scheduler preemptivo de prioridad fija
implementado enteramente en hardware (consistente con el Principio 6
— no existe un scheduler *programable*). Como en cualquier esquema
así, **`C0` no tiene ninguna garantía de forward-progress** bajo carga
sostenida de eventos de mayor prioridad — es un trade-off aceptado, no
un defecto, y conviene decirlo explícitamente en vez de dejarlo
implícito bajo la palabra "determinista".

### Frontera de preemption y operaciones multiciclo

Las operaciones multiciclo (loads pendientes, operaciones de ALU
multiciclo) no se preemptan a mitad de camino: mientras el pipe está
en stall, no hay ninguna ranura de fetch libre para otro contexto, así
que la preemption queda diferida naturalmente hasta el próximo límite
de instrucción. Esto da un comportamiento de "frontera de interrupción
precisa" sin hardware adicional.

Los puertos de memoria de instrucciones y de datos son físicamente
separados. El puerto I lo maneja únicamente la etapa de fetch del
contexto entrante; el puerto D lo maneja únicamente la operación de
memoria del contexto saliente que todavía está drenando en EX/WB. Como
solo hay un contexto en el pipe a la vez, estos dos accesos pueden
solaparse de forma segura durante un cambio de contexto sin lógica de
arbitraje adicional, y ninguna transacción de memoria queda nunca a
medio completar por una preemption.

------------------------------------------------------------------------

## 8. CSRs propuestos

La asignación concreta de direcciones vive en el spec formal
([Xhctx-spec.md](Xhctx-spec.md), Capítulo 3): `mctxctl`/`mevent`/
`meventdata` en el rango custom R/W de modo M (`0x7C0`–`0x7C2`,
provisorio), y `meventdata[ctx_id]`/`mepc[ctx_id]`/`mcause[ctx_id]`
vía acceso indirecto `Smcsrind` (`miselect`/`mireg`) en vez de una
dirección de CSR cruda por elemento — 48 direcciones habrían agotado
casi todo el rango custom de solo lectura de modo M. `Xhctx` pasa a
depender de `Smcsrind` (extensión estándar, ratificada 2024).

El acceso a todos los CSRs de esta sección está restringido a modo
**M** en la binding de referencia (un RV32E chico sin S-mode) — los
rangos custom de M y S son direcciones físicamente distintas en
RISC-V, así que "M o S" no es un interruptor en runtime sino una
elección de qué rango asignar; una binding en modo S queda fuera de
alcance de esta primera versión.

No existe un CSR `mctx` separado: como solo el contexto que está
efectivamente ejecutando puede ejecutar la instrucción que lo leería,
"el contexto actualmente en ejecución" es siempre, tautológicamente,
el propio `mhartid` del lector. `mhartid` (Sección 3) cumple ese rol
sin necesidad de un CSR nuevo.

### Lectura cruzada entre contextos (`[ctx_id]`)

`meventdata[ctx_id]`, `mepc[ctx_id]` y `mcause[ctx_id]` son de solo
lectura para **cualquier** `ctx_id`, no solo el propio — esto es una
desviación deliberada del aislamiento estricto de CSR entre harts que
asume RISC-V estándar. Se justifica con el mismo precedente que ya usa
la extensión Hypervisor de RISC-V, donde HS-mode lee/escribe el estado
sombreado de VS-mode (`vsepc`, `vscause`, etc.): contextos fuertemente
acoplados que comparten un recurso de ejecución pueden exponer una
ventana de introspección controlada sobre el estado del otro, sin que
eso implique que dejan de ser harts independientes en todo lo demás.

### mevent (RO/WO)

-   Lectura: `event_id` que activó al contexto actualmente en
    ejecución.
-   Escritura: especifica el `event_id` de destino (de la tabla
    general o del rango `SW_TARGET_BASE`, Sección 5) y dispara el push
    a la FIFO correspondiente, sujeto a la regla de bypass/prioridad
    de la Sección 7.

### meventdata (RO/WO, bankeado)

-   **Escritura sin índice** — la única forma escribible. Siempre
    direcciona implícitamente el slot bankeado del contexto que está
    *ejecutando en ese momento*; no existe ninguna instrucción capaz
    de escribir el slot de otro contexto.
-   **Lectura indexada `meventdata[ctx_id]`** — de solo lectura, para
    cualquier `ctx_id`. Se usa para introspección (debug, o un handler
    inspeccionando el dato de evento que tiene otro contexto).

### mctxctl (WO)

CSR de control utilizado para abandonar el contexto actual.

### mepc[ctx_id] (RO, bankeado)

Un registro por contexto con el PC de falla de ese contexto. Lo
escribe únicamente el hardware, en el momento en que un contexto es
forzado a abandonar por una excepción síncrona (ver Sección 10, Caso
D). No es un alias del banco de PC arquitectónico en vivo — es un
registro dedicado, para evitar una carrera donde el contexto que falló
podría ser reactivado por un evento no relacionado antes de que el
handler lea su estado de falla congelado.

### mcause[ctx_id] (RO, bankeado)

Un registro por contexto con el código de causa de la excepción
síncrona más reciente de ese contexto. Misma semántica de escritura
que `mepc[ctx_id]`.

### mstatus.MIE — pausa de preemption

El bit `MIE` de `mstatus`, ya estándar y bankeado por hart, se
reutiliza como mecanismo para que un contexto proteja una secuencia
read-modify-write de varias instrucciones contra un periférico
compartido con otro contexto (Sección 7) — un hazard que el banking de
estado arquitectónico no cubre, porque el estado en riesgo vive
*afuera* del contexto, en el dispositivo. Con `MIE = 0`, la preemption
por prioridad (Sección 7) no desaloja al contexto activo.

**`MIE = 0` no enmascara la preemption forzada por timer/watchdog**
(Sección 7) — si lo hiciera, un contexto podría des-habilitarla para
siempre y reintroducir el livelock que ese mecanismo existe para
evitar. El timer/watchdog es, en ese sentido, no-enmascarable, análogo
a una NMI.

------------------------------------------------------------------------

## 9. Eventos software

El software genera eventos escribiendo:

``` text
meventdata
mevent
```

La escritura de `mevent` captura el payload previamente escrito en
`meventdata` y lo inyecta en el Event Fabric, dirigido al `event_id`
recién escrito.

Como `meventdata` (sin índice) solo puede direccionar el slot bankeado
del contexto que ejecuta la escritura, y ningún otro contexto puede
tocar ese slot, la secuencia de dos instrucciones es segura bajo
preemption total: si el contexto emisor es desalojado entre las dos
escrituras (Sección 7), ningún otro contexto puede corromper el
payload ya guardado, y al retomar encuentra su propio dato intacto.

Para direccionar un contexto destino arbitrario (no solo las fuentes
fijas de la tabla de ruteo), el software escribe en `mevent` un
`event_id` del rango reservado `SW_TARGET_BASE + ctx_id` (Sección 5),
de ruteo identidad. Este es el mismo mecanismo que se usa para
inicializar los contextos en el arranque (Sección 11).

No existe una instrucción SEV; la emisión de eventos forma parte de la
interfaz CSR.

------------------------------------------------------------------------

## 10. Abandono del contexto

El abandono del contexto no utiliza `WFI`.

Se realiza mediante una escritura a `mctxctl`. Este es el disparador
**voluntario** de cambio de contexto; existe además un disparador
**involuntario** (preemption por prioridad, Sección 7), que reutiliza
el mismo mecanismo de squash-and-redirect de 1 ciclo (Sección 3) pero
puede ocurrir en cualquier ciclo, no solo al ejecutar `mctxctl`.

### Caso A --- Hay eventos pendientes del mismo contexto

-   Consumir siguiente evento.
-   Actualizar `mevent`.
-   Continuar ejecutando el mismo contexto.
-   No cambiar `active_ctx` (Sección 3).

Equivale funcionalmente a un NOP respecto al cambio de contexto.

### Caso B --- Hay eventos pendientes en otros contextos

-   El contexto actual pasa a espera.
-   El Event Fabric selecciona el contexto de mayor prioridad.
-   Se actualiza `active_ctx`.

### Caso C --- No existen eventos pendientes

-   El procesador entra en estado Idle/Sleep.
-   El siguiente evento reactiva el contexto correspondiente.

### Caso D --- Excepción síncrona

Las excepciones síncronas se resuelven generando un evento, no
manteniendo el trap estándar de RISC-V ni quedando locales al
contexto:

-   El contexto que falla es forzado a abandonar (disparador de
    hardware, no una escritura de `mctxctl`).
-   El hardware escribe `mepc[ctx_id]` y `mcause[ctx_id]` (Sección 8)
    del contexto que falló.
-   Se empuja una entrada a la FIFO del contexto de excepción
    (dedicado, definido por implementación) con `event_data = mectx`
    (el id del contexto que falló).
-   El handler lee `mectx` de su evento consumido, y con eso indexa
    `mepc[mectx]` / `mcause[mectx]` para obtener el detalle de la
    falla.

------------------------------------------------------------------------

## 11. Estado mínimo del contexto

Evitar una máquina de estados estilo RTOS.

Idealmente únicamente existe:

``` text
waiting[15:0]
```

Los demás estados son implícitos:

-   RUNNING → `ctx == active_ctx`
-   WAITING → `waiting = 1`
-   SUSPENDED → `ctx != active_ctx && waiting = 0`

No introducir READY, BLOCKED o TERMINATED salvo necesidad demostrada.

Estado adicional, fuera de esta máquina implícita: `mepc[ctx_id]` /
`mcause[ctx_id]` (Sección 8, poblados solo ante excepción) y, para
debug (Sección 14), un bit de máscara por contexto que lo excluye del
despacho sin afectar al resto.

### Secuencia de arranque

El reset solo activa un único contexto de reset definido por
implementación (`ctx0` por convención): `waiting[15:1] = 0`, `ctx0`
corriendo desde el PC de reset. Todos los demás contextos arrancan
suspendidos con `waiting = 0`. La inicialización de `ctx1..ctx15` se
hace enteramente por software, con `ctx0` direccionando a cada uno a
través del mecanismo de eventos estándar (Sección 9, rango
`SW_TARGET_BASE`) — no hace falta ningún camino de inicialización de
hardware dedicado más allá de levantar el único contexto de reset.

------------------------------------------------------------------------

## 12. Modelo de programación

Cada contexto se implementa como un actor persistente.

Ejemplo conceptual:

``` c
void audio_context(void)
{
    while (1) {
        process_audio(meventdata);

        abandon_context();
    }
}
```

No existen:

-   ISR
-   Scheduler software
-   Context switch software
-   mret

------------------------------------------------------------------------

## 13. Compatibilidad con RISC-V

La extensión debe preservar:

-   RV32E
-   ABI
-   ELF
-   Toolchain C

Se trata de una **extensión experimental** y no de una ISA
independiente, nombrada **`Xhctx`** ("Hart Context"), siguiendo la
convención de prefijo `X` de RISC-V para extensiones no estándar.

La especificación deberá redactarse con formato RISC-V International.

### Destino de `mret` y `wfi`

Ambas son instrucciones válidas del ISA base que un compilador o
código heredado puede emitir; su comportamiento bajo `Xhctx` queda
definido explícitamente en vez de dejarse indefinido:

-   **`mret`** no puede ejecutarse en este modelo de sistema — se
    trata como instrucción ilegal y dispara el Caso D de la Sección 10
    (excepción síncrona, entregada como evento al contexto de
    excepción). No hay ningún flujo de programa válido bajo `Xhctx`
    que deba ejecutarla.
-   **`wfi`** es un **NOP** — no tiene efecto sobre el scheduling
    (Sección 7 ya cubre el caso de "no hay más trabajo" mediante el
    Caso C de abandono). Se define así, en vez de tratarla como
    ilegal, para no romper binarios que la emitan como parte de un
    idle loop convencional.

------------------------------------------------------------------------

## 14. Implementación de referencia

Objetivo RTL:

-   RV32E
-   Pipeline de 3 etapas
-   16 contextos
-   Register File completamente bankeado
-   Pipeline completamente bankeado
-   16 FIFOs
-   Priority Encoder
-   Bypass de eventos
-   Activación de contexto en un ciclo objetivo
-   Tag de `ctx_id` propagado por IF/ID, ID/EX y EX/WB
-   Bancos `mepc[ctx_id]` / `mcause[ctx_id]`
-   Bit de máscara por contexto para el Priority Encoder (soporte de
    debug: permite excluir un contexto del despacho sin detener el
    core físico)
-   Cada contexto expuesto como hart independiente ante el Debug
    Module estándar de RISC-V, reutilizando su infraestructura
    multi-hart existente

------------------------------------------------------------------------

## 15. Entregables esperados

Claude Code deberá generar toda la documentación en **inglés** siguiendo
este orden:

1.  ISA Overview
2.  Motivation
3.  Architectural Model
4.  Programmer Model
5.  Event Model
6.  Context Lifecycle
7.  CSR Specification
8.  Event Routing
9.  Arbitration Rules
10. Timing Diagrams
11. RV32E Reference Microarchitecture
12. RTL Skeleton
13. Verification Strategy
14. Area & Timing Estimates
15. Comparison with Prior Art
