# Proyecto Nios II: LEDs controlados mediante interrupciones

**Curso:** Estructura del Computador  
**Sistema:** DE2-115 Computer for Nios II  
**Proyecto:** `ejemplo2.amp`  
**Base del análisis:** archivos suministrados por el usuario.

## 1. Objetivo y estado del proyecto

El proyecto pretende mostrar un patrón en los LEDs y rotarlo periódicamente mediante interrupciones de un temporizador. Dos pulsadores permiten modificar su comportamiento:

| Evento | Acción prevista |
| --- | --- |
| Interrupción del temporizador | Mostrar el patrón actual y preparar la siguiente rotación |
| Evento de `KEY[0]` | Cargar en el patrón el valor de los interruptores `SW` |
| Evento de `KEY[1]` | Cambiar la dirección de rotación |

El programa principal configura el hardware y permanece en un ciclo. Las rutinas de servicio de interrupción —ISR, *Interrupt Service Routine*— realizan el trabajo cuando se produce un evento.

**Problema encontrado en los archivos entregados:** faltan las instrucciones `ret` al final de las dos ISR y `eret` al final del manejador de excepciones. Por ello, el código recibido no completa correctamente los retornos. Las secciones siguientes explican su diseño previsto y, al final, muestran las correcciones mínimas necesarias. Los archivos fuente originales no se han modificado.

## 2. Archivos y responsabilidades

| Archivo | Responsabilidad |
| --- | --- |
| `ejemplo2.amp` | Describe el proyecto del Intel FPGA Monitor Program: arquitectura, sistema, archivos y secciones de memoria |
| `interrupt_example.s` | Inicializa pila, temporizador e interrupciones; declara las variables compartidas |
| `exception_handler.s` | Contiene el código de reset y el manejador que identifica la fuente de interrupción |
| `interval_timer_ISR.s` | Atiende el temporizador y actualiza el patrón rotatorio |
| `pushbutton_ISR.s` | Atiende los eventos de pulsadores y cambia las variables compartidas |
| `globals.s` | Define las constantes `LEFT` y `RIGHT` |
| `address_map_nios2.s` | Define las direcciones de memoria y periféricos |
| `makefile` | Ensambla, enlaza y genera los archivos de salida |


## 3. Diferencia respecto al ejemplo anterior

| Aspecto | Ejemplo `getting_started.s` | Proyecto actual |
| --- | --- | --- |
| Actualización de LEDs | En el ciclo principal | En la ISR del temporizador |
| Control del intervalo | Ciclo de decremento | Temporizador hardware |
| Detección de pulsadores | Consulta repetida del puerto | Interrupción por evento capturado |
| Dirección del movimiento | Rotación a la izquierda | Derecha o izquierda |
| Carga desde `SW` | Construye un patrón con rotaciones y OR | Copia directamente la palabra leída |
| Trabajo del programa principal | Lectura, escritura y retardo | Configuración inicial y ciclo `IDLE` |

El programa actual sigue consumiendo ejecución en `IDLE`: no utiliza una instrucción de reposo. Su ventaja didáctica es que el temporizador produce los eventos periódicos y las ISR concentran la atención a los periféricos.

## 4. Direcciones y registros de periféricos

Las direcciones base utilizadas son:

| Símbolo | Dirección | Periférico |
| --- | --- | --- |
| `LED_BASE` | `0xFF200000` | LEDs; coincide con `LEDR_BASE` |
| `SW_BASE` | `0xFF200040` | Interruptores |
| `KEY_BASE` | `0xFF200050` | Pulsadores |
| `TIMER_BASE` | `0xFF202000` | Temporizador |

El programa utiliza distintos registros internos de cada periférico mediante desplazamientos respecto a su dirección base.

| Periférico | Desplazamiento | Dirección | Uso en el código |
| --- | --- | --- | --- |
| Temporizador | `0x0` | `0xFF202000` | Estado; limpiar la condición de interrupción |
| Temporizador | `0x4` | `0xFF202004` | Control |
| Temporizador | `0x8` | `0xFF202008` | Mitad inferior del período |
| Temporizador | `0xC` | `0xFF20200C` | Mitad superior del período |
| Pulsadores | `0x8` | `0xFF200058` | Máscara de interrupciones |
| Pulsadores | `0xC` | `0xFF20005C` | Captura de flancos, *edge capture* |

Estas direcciones deben corresponder al sistema realmente configurado en la FPGA.

## 5. Variables compartidas y constantes

En `globals.s` aparecen:

```assembly
.equ LEFT,  0
.equ RIGHT, 1
```

Son constantes del ensamblador; no reservan memoria.

En la sección `.data` de `interrupt_example.s` se definen:

```assembly
.global PATTERN
PATTERN:
.word 0x0F0F0F0F

.global SHIFT_DIR
SHIFT_DIR:
.word RIGHT
```

| Variable | Valor inicial | Significado |
| --- | --- | --- |
| `PATTERN` | `0x0F0F0F0F` | Palabra de 32 bits que se enviará a los LEDs |
| `SHIFT_DIR` | `1` | Dirección inicial: derecha |

`.global` permite que otros archivos objeto utilicen esos símbolos durante el enlazado. Las ISR los referencian mediante `.extern`.

`PATTERN` se modifica tanto en la ISR del temporizador como en la de pulsadores. `SHIFT_DIR` se modifica en la ISR de pulsadores y se consulta en la del temporizador.

## 6. Arranque y distribución de memoria

### 6.1. Sección de reset

En `exception_handler.s`:

```assembly
.section .reset, "ax"
    movia r2, _start
    jmp   r2
```

La sección contiene un salto al programa principal. `a` indica que la sección ocupa memoria de ejecución y `x` que contiene código ejecutable.

### 6.2. Ubicaciones configuradas en el proyecto

El archivo `.amp` asigna:

| Sección | Rango configurado |
| --- | --- |
| `.reset` | `0x00000000`–`0x0000001F` |
| `.exceptions` | `0x00000020`–`0x000001FF` |
| `.text` | A partir de `0x00000200`, dentro de SDRAM |

La sección `.exceptions` debe quedar en la dirección de excepción configurada para el procesador. No basta con ponerle ese nombre: el enlazado y el hardware deben coincidir.

El proyecto también declara `_start` como símbolo de entrada. Según el mecanismo de inicio utilizado, la ejecución puede comenzar en ese símbolo o llegar allí desde el vector de reset.

## 7. Programa principal: `interrupt_example.s`

### Paso 1. Incluir las definiciones

```assembly
.include "address_map_nios2.s"
.include "globals.s"
```

Se incorporan las direcciones y las constantes de dirección de movimiento.

### Paso 2. Inicializar la pila

```assembly
movia sp, SDRAM_END - 3
```

Con `SDRAM_END = 0x07FFFFFF`:

```text
sp = 0x07FFFFFC
```

La dirección queda alineada a cuatro bytes. La pila crecerá hacia direcciones menores: las rutinas restan al puntero antes de guardar registros.

La pila es indispensable porque las interrupciones deben preservar el estado del código interrumpido.

### Paso 3. Cargar la dirección del temporizador

```assembly
movia r16, TIMER_BASE
```

`r16` pasa a contener `0xFF202000`.

### Paso 4. Programar el período

```assembly
movia r12, 5000000
sthio r12, 8(r16)
srli  r12, r12, 16
sthio r12, 0xC(r16)
```

El valor decimal 5 000 000 equivale a `0x004C4B40`. El periférico recibe el período en dos mitades:

| Registro | Valor escrito |
| --- | --- |
| Período inferior | `0x4B40` |
| Período superior | `0x004C` |

`sthio` escribe los 16 bits inferiores del registro mediante un acceso de E/S. `srli` desplaza lógicamente el dato 16 posiciones a la derecha para colocar la mitad superior en esos bits inferiores.

El comentario del código supone un reloj del temporizador de 100 MHz:

\[
T \approx \frac{5\,000\,000}{100\,000\,000}=0.05\ \text{s}=50\ \text{ms}
\]

Esto corresponde aproximadamente a 20 actualizaciones por segundo. La frecuencia debe ser la del **temporizador**, que no tiene por qué coincidir con la del procesador en otro sistema. El intervalo exacto también depende de la convención de conteo del periférico, incluido el posible ciclo adicional asociado al conteo hasta cero.

### Paso 5. Iniciar el temporizador

```assembly
movi  r15, 0b0111
sthio r15, 4(r16)
```

El valor de control establece:

| Bit | Nombre | Valor | Función |
| --- | --- | --- | --- |
| 0 | `ITO` | 1 | Habilitar la generación de interrupciones |
| 1 | `CONT` | 1 | Operación continua |
| 2 | `START` | 1 | Iniciar el conteo |
| 3 | `STOP` | 0 | No solicitar parada |

El temporizador seguirá generando eventos periódicos sin que el programa principal vuelva a cargar el contador en cada vuelta.

### Paso 6. Habilitar eventos de `KEY[0]` y `KEY[1]`

```assembly
movia r15, KEY_BASE
movi  r7, 0b11
stwio r7, 8(r15)
```

Se habilitan los bits 0 y 1 en la máscara del puerto de pulsadores. Los demás bits permanecen deshabilitados.

Esto habilita fuentes **dentro del periférico**. Todavía deben habilitarse la línea correspondiente y la aceptación global en el procesador.

### Paso 7. Habilitar las líneas IRQ del procesador

```assembly
movia r7, 0x00000001
movia r8, 0x00000002
or    r7, r7, r8
wrctl ienable, r7
```

El resultado es `0x00000003`:

| Bit de `ienable` | Fuente que supone este proyecto |
| --- | --- |
| 0 | Temporizador, IRQ 0 |
| 1 | Puerto de pulsadores, IRQ 1 |

La asignación de IRQ debe coincidir con el sistema hardware.

### Paso 8. Habilitar globalmente las interrupciones

```assembly
movi  r7, 1
wrctl status, r7
```

Se activa el bit global `PIE` del registro `status`. El código escribe el valor completo `1` en ese registro; para otro contexto podría ser necesario conservar otros bits.

En este ejemplo hay tres habilitaciones que deben coincidir:

1. La del periférico: `ITO` o máscara de pulsadores.
2. La de la línea IRQ en `ienable`.
3. La global `PIE` en `status`.

### Paso 9. Permanecer en el ciclo principal

```assembly
IDLE:
    br IDLE
```

El procesador repite el salto. Cuando acepta una interrupción, transfiere el control al manejador de excepciones y, con los retornos corregidos, después reanuda el programa.

## 8. Manejador de excepciones: `exception_handler.s`

### Paso 1. Reservar espacio y preservar `et`

```assembly
subi sp, sp, 16
stw  et, 0(sp)
```

Se reservan cuatro palabras de pila. `et` es el alias del registro `r24`, utilizado aquí como temporal del manejador.

### Paso 2. Consultar interrupciones pendientes

```assembly
rdctl et, ctl4
beq   et, r0, SKIP_EA_DEC
```

`ctl4` es el registro de control `ipending`. Sus bits permiten identificar las líneas de interrupción pendientes y habilitadas.

Este manejador interpreta un valor distinto de cero como una interrupción externa. Es una lógica sencilla para este ejemplo; no constituye un clasificador general de todas las causas de excepción.

### Paso 3. Ajustar la dirección de retorno

```assembly
subi ea, ea, 4
```

Para el tratamiento de interrupciones externas empleado aquí, se resta una instrucción de cuatro bytes a `ea`, el registro de dirección de retorno de excepción (`r29`). Así se prepara el punto desde el que debe continuar la ejecución al retornar.

El código omite ese ajuste cuando `ipending` es cero. No debe aplicarse indiscriminadamente a todas las excepciones síncronas.

### Paso 4. Guardar el resto del contexto

```assembly
stw ea, 4(sp)
stw ra, 8(sp)
stw r22, 12(sp)
```

| Desplazamiento | Registro guardado |
| --- | --- |
| `0(sp)` | `et` |
| `4(sp)` | `ea` |
| `8(sp)` | `ra` |
| `12(sp)` | `r22` |

`ra` debe preservarse porque `call` lo modifica para guardar la dirección de retorno de una subrutina.

### Paso 5. Seleccionar la ISR

```assembly
rdctl et, ctl4
bne   et, r0, CHECK_LEVEL_0
```

Si no hay IRQ pendiente, el bloque `NOT_EI` salta al final. El archivo reconoce que no implementa la atención a instrucciones no implementadas o a `trap`. No puede considerarse un manejador completo para esos casos.

Para el temporizador:

```assembly
CHECK_LEVEL_0:
    andi r22, et, 0b1
    beq  r22, r0, CHECK_LEVEL_1
    call INTERVAL_TIMER_ISR
    br   END_ISR
```

Para los pulsadores:

```assembly
CHECK_LEVEL_1:
    andi r22, et, 0b10
    beq  r22, r0, END_ISR
    call PUSHBUTTON_ISR
```

La operación `andi` aísla el bit de interés.

**Si ambas fuentes están pendientes**, se atiende primero el temporizador y se salta al final. La interrupción de pulsadores queda pendiente y podrá atenderse después del retorno. Esta preferencia surge del orden del código; las etiquetas `LEVEL_0` y `LEVEL_1` no configuran por sí mismas un sistema de prioridades o anidamiento.

### Paso 6. Restaurar y retornar

El archivo restaura los cuatro registros y libera los 16 bytes:

```assembly
ldw  et, 0(sp)
ldw  ea, 4(sp)
ldw  ra, 8(sp)
ldw  r22, 12(sp)
addi sp, sp, 16
```

**Después falta `eret`.** Esa instrucción realiza el retorno de excepción, utiliza `ea` y restaura el estado de ejecución correspondiente desde `estatus`.

La directiva `.end` no retorna ni detiene el procesador.

## 9. ISR del temporizador: `interval_timer_ISR.s`

### Paso 1. Guardar los registros

La rutina reserva 40 bytes y guarda diez registros: `ra`, `r4`, `r5`, `r6`, `r8`, `r10`, `r20`, `r21`, `r22` y `r23`.

Algunos no se modifican en el cuerpo actual, pero se conservan en la plantilla entregada.

### Paso 2. Reconocer el evento

```assembly
movia r10, TIMER_BASE
sthio r0, 0(r10)
```

Se escribe cero en el registro de estado para limpiar la condición de timeout del temporizador utilizado por el ejemplo. Sin limpiar la fuente, la solicitud podría permanecer activa y provocar nuevas entradas al manejador.

Reconocer el evento no significa deshabilitar las futuras interrupciones periódicas.

### Paso 3. Mostrar el patrón actual

```assembly
movia r20, LED_BASE
movia r21, PATTERN
movia r22, SHIFT_DIR
ldw   r6, 0(r21)
stwio r6, 0(r20)
```

`ldw` lee la variable de memoria ordinaria. `stwio` escribe el periférico.

### Paso 4. Consultar la dirección

```assembly
ldw  r5, 0(r22)
movi r8, RIGHT
bne  r5, r8, SHIFT_L
```

Si `SHIFT_DIR` vale `RIGHT`, se rota a la derecha; cualquier valor distinto toma el camino izquierdo. Los valores previstos son únicamente cero y uno.

### Paso 5. Rotar un bit

```assembly
SHIFT_R:
    movi r5, 1
    ror  r6, r6, r5
    br   STORE_PATTERN

SHIFT_L:
    movi r5, 1
    rol  r6, r6, r5
```

`ror` y `rol` rotan los **32 bits**. El bit que sale por un extremo entra por el otro. Aunque algunos comentarios dicen *shift*, las instrucciones implementan rotación circular.

### Paso 6. Guardar el siguiente patrón

```assembly
STORE_PATTERN:
    stw r6, 0(r21)
```

El orden es fundamental:

1. Se muestra el patrón actual.
2. Se calcula la siguiente posición.
3. Se guarda en `PATTERN` para el próximo evento.

Por tanto, después de la ISR, la variable puede contener un valor diferente del que todavía muestran los LEDs.

### Paso 7. Restaurar y retornar

La rutina recupera los diez registros y suma 40 a `sp`. **Falta `ret`** para volver a la instrucción posterior al `call` del manejador.

### Ejemplo de rotación inicial a la derecha

| Evento del temporizador | Valor enviado a LEDs | `PATTERN` guardado para el siguiente evento |
| --- | --- | --- |
| 1 | `0x0F0F0F0F` | `0x87878787` |
| 2 | `0x87878787` | `0xC3C3C3C3` |
| 3 | `0xC3C3C3C3` | `0xE1E1E1E1` |
| 4 | `0xE1E1E1E1` | `0xF0F0F0F0` |

El ancho real del puerto de LEDs determina qué bits se observan físicamente.

## 10. ISR de pulsadores: `pushbutton_ISR.s`

### Paso 1. Guardar el contexto local

La rutina reserva 20 bytes y guarda `ra`, `r10`, `r11`, `r12` y `r13`.

### Paso 2. Leer los eventos capturados

```assembly
movia r10, KEY_BASE
ldwio r11, 0xC(r10)
stwio r11, 0xC(r10)
```

Se lee el registro de captura de flancos y luego se escribe para reconocer el evento.

Este registro conserva la información de los flancos detectados; no es simplemente el estado instantáneo de los pulsadores. El flanco que genera un evento y la semántica exacta de limpieza dependen de la configuración del puerto PIO. La secuencia del ejemplo supone que esa escritura limpia las capturas correspondientes; debe comprobarse al reutilizarlo con otro hardware.

### Paso 3. Detectar `KEY[0]`

```assembly
andi r13, r11, 0b0001
beq  r13, zero, CHECK_KEY1
```

`zero` es otro nombre de `r0`. Si el bit cero no está activo, se omite la carga de un nuevo patrón.

Si está activo:

```assembly
movia r10, SW_BASE
ldwio r12, 0(r10)
movia r10, PATTERN
stw   r12, 0(r10)
```

Se leen los interruptores en el momento de atender la ISR y se copia la palabra directamente a `PATTERN`.

**No se repite el byte en las cuatro posiciones**, como ocurría en el ejemplo anterior. Si la lectura es `0x00000001`, la rotación es sobre esa palabra de 32 bits:

```text
0x00000001 → 0x80000000 → 0x40000000 → 0x20000000 → …
```

Si esos bits superiores no tienen LEDs asociados, habrá posiciones en las que el bit encendido no se vea. El ciclo completo de un único bit requiere 32 eventos.

Además, la ISR de pulsadores no escribe el puerto LED. El nuevo patrón será visible cuando lo muestre la ISR del temporizador, normalmente en el siguiente evento periódico.

### Paso 4. Detectar `KEY[1]`

```assembly
CHECK_KEY1:
    andi r13, r11, 0b0010
    beq  r13, zero, END_PUSHBUTTON_ISR
```

Si el bit uno está activo:

```assembly
movia r10, SHIFT_DIR
ldw   r12, 0(r10)
xori  r12, r12, 1
stw   r12, 0(r10)
```

`xori` invierte el bit inferior:

| Valor anterior | Operación | Nuevo valor |
| --- | --- | --- |
| `0`, izquierda | `0 XOR 1` | `1`, derecha |
| `1`, derecha | `1 XOR 1` | `0`, izquierda |

La operación funciona como alternancia porque la variable se mantiene en cero o uno.

Cambiar la dirección no recalcula el patrón ya preparado por la ISR anterior. La siguiente ISR mostrará ese patrón y lo rotará usando la nueva dirección.

### Paso 5. Considerar eventos simultáneos

Las comprobaciones son consecutivas, no excluyentes. Si los bits 0 y 1 están activos en la misma lectura, la ISR carga el patrón de `SW` y también alterna la dirección.

La rutina no implementa antirrebote por software. El número de eventos observados dependerá también del acondicionamiento y la configuración del hardware.

### Paso 6. Restaurar y retornar

Se recuperan los cinco registros y se liberan 20 bytes. **Falta `ret`** para volver al manejador.

## 11. Uso de pila y conservación del estado

Con los retornos corregidos, el consumo explícito de pila de cada camino es:

| Camino | Manejador | ISR | Total |
| --- | --- | --- | --- |
| Temporizador | 16 bytes | 40 bytes | 56 bytes |
| Pulsadores | 16 bytes | 20 bytes | 36 bytes |

`call` guarda el retorno en `ra`; no apila automáticamente todos los registros. Las instrucciones `stw` y `ldw` de las rutinas implementan esa conservación.

El código no vuelve a habilitar `PIE` dentro del manejador, por lo que no implementa interrupciones externas anidadas. Durante la atención, otras fuentes pueden quedar pendientes. Las dos ISR no se intercalan entre sí en el flujo previsto.

## 12. Correcciones mínimas de retorno

### 12.1. Final de `interval_timer_ISR.s`

Después de la restauración existente:

```assembly
    addi sp, sp, 40
    ret
.end
```

### 12.2. Final de `pushbutton_ISR.s`

Después de la restauración existente:

```assembly
    addi sp, sp, 20
    ret
.end
```

### 12.3. Final de `exception_handler.s`

Después de la restauración existente:

```assembly
    addi sp, sp, 16
    eret
.end
```

| Instrucción | Destino y función |
| --- | --- |
| `ret` | Regresa usando `ra` desde una subrutina llamada con `call` |
| `eret` | Retorna de una excepción usando `ea` y restablece el estado de excepción |

No son intercambiables. Sin ellas, el procesador continúa leyendo las instrucciones que encuentre después del bloque; el resultado dependerá de la disposición final del ejecutable y no seguirá el flujo previsto.

Estas inserciones corrigen los retornos ausentes. No convierten el manejador en una implementación general para toda clase de excepciones.

## 13. Construcción del ejecutable

El `makefile` fue generado por el Intel FPGA Monitor Program e indica que no debe modificarse manualmente.

La construcción tiene tres etapas:

| Herramienta | Entrada | Salida o función |
| --- | --- | --- |
| `nios2-elf-as` | Fuentes `.s` | Objetos como `interrupt_example.s.o` |
| `nios2-elf-ld` | Objetos y script de enlace | `interrupt_example.elf` |
| `nios2-elf-objcopy` | Archivo ELF | `interrupt_example.srec` |

El enlazador resuelve símbolos como `PATTERN`, `SHIFT_DIR` y los nombres de las ISR. También coloca `.reset` en `0x0` y `.exceptions` en `0x20`.

El makefile contiene rutas de una instalación de Intel FPGA Lite 18.1 en Windows. En otro equipo deben ajustarse las opciones del proyecto y regenerarse los archivos de construcción según corresponda. No se debe asumir que esas rutas existen.

`globals.s` figura como fuente y también se incorpora con `.include`. Su contenido actual solo define constantes `.equ`; no contiene variables compartidas ni código ejecutable.

El archivo `.amp` tiene el cable como `null` y la placa como `Not Required`. Para trabajar con hardware físico hay que seleccionar la conexión y cargar un sistema compatible con el proyecto.

## 14. Mejoras de inicialización a considerar

Además de los retornos obligatorios, conviene evaluar estas mejoras para una versión más robusta:

- Limpiar capturas antiguas de pulsadores antes de habilitar sus interrupciones.
- Limpiar el estado pendiente del temporizador antes de ponerlo en marcha.
- Mantener deshabilitada la aceptación global mientras se completa la configuración.
- Decidir explícitamente si el patrón debe rotar sobre 32 bits o sobre el número de LEDs utilizados.
- Implementar antirrebote si el hardware no lo proporciona y la aplicación lo requiere.
- Añadir atención real a las demás excepciones si el programa deja de ser un ejemplo limitado.

Son mejoras distintas del problema de los retornos y no están implementadas en los archivos recibidos.

## 15. Secuencia de comprobación en clase

Una vez añadidos los retornos y verificada la correspondencia con el hardware:

1. Ensamblar y enlazar el proyecto.
2. Comprobar que `SHIFT_DIR` comienza en `RIGHT` y `PATTERN` en `0x0F0F0F0F`.
3. Ejecutar y observar la actualización periódica de LEDs.
4. Activar `KEY[1]` y observar el cambio de dirección.
5. Establecer un patrón sencillo en `SW`, generar un evento de `KEY[0]` y observar su carga.
6. Seguir en el depurador las variables `PATTERN` y `SHIFT_DIR`.
7. Comprobar cómo el registro de LEDs conserva el patrón mostrado mientras `PATTERN` ya contiene el siguiente.
8. Observar la pila al entrar y salir del manejador y de cada ISR; `sp` debe recuperar su valor inicial.

La interpretación de «pulsar» o «soltar» como evento dependerá del flanco configurado en el puerto.

## 16. Preguntas para los estudiantes

1. ¿Por qué habilitar `ienable` no basta para recibir interrupciones?
2. ¿Qué diferencia hay entre el estado de un pulsador y su registro de captura de flancos?
3. ¿Por qué debe limpiarse la condición de interrupción en el periférico?
4. ¿Por qué el manejador guarda `ra` antes de llamar a una ISR?
5. ¿Qué ocurriría si la ISR modificara un registro sin restaurarlo?
6. ¿Por qué la ISR termina con `ret` y el manejador con `eret`?
7. ¿Qué fuente se atiende primero si temporizador y pulsadores están pendientes simultáneamente?
8. ¿Por qué un único bit puede desaparecer de los LEDs durante parte de la rotación?
9. ¿Cuál es la diferencia entre el valor mostrado en LEDs y el valor guardado en `PATTERN` al terminar la ISR del temporizador?
10. ¿Cómo cambiaría el intervalo aproximado si el reloj del temporizador fuera de 50 MHz con la misma constante?
11. ¿Por qué el ciclo `IDLE` no equivale a poner el procesador en reposo?
12. ¿Qué información habría que conocer del sistema hardware para trasladar el ejemplo a otra configuración?

## 17. Alcance de la revisión

Se revisaron los archivos fuente, la descripción `.amp`, el makefile y la igualdad del mapa de direcciones con el ejemplo anterior. Las trazas numéricas se derivan de las operaciones del programa.

No se ensambló con la herramienta Nios II ni se ejecutó sobre la placa. El comportamiento descrito como previsto requiere añadir los tres retornos ausentes y utilizar una configuración hardware compatible.
