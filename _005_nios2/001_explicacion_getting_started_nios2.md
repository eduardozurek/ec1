# Explicación paso a paso de `getting_started.s`

**Curso:** Estructura del Computador — Nios II y entrada/salida  
**Archivos analizados:** `getting_started.s` y `address_map_nios2.s`  
**Sistema indicado por el mapa de direcciones:** DE2-115 Computer

## 1. Propósito del programa

El programa muestra un patrón de bits en los LEDs y lo rota continuamente hacia la izquierda. Cuando detecta un pulsador presionado, lee los interruptores `SW`, construye un nuevo patrón y espera a que se liberen todos los pulsadores. Después muestra el nuevo patrón y continúa la rotación.

Este ejemplo permite estudiar:

- Entrada/salida mapeada en memoria.
- Diferencia entre una dirección y el dato almacenado en ella.
- Lectura de interruptores y pulsadores.
- Escritura en LEDs.
- Operaciones lógicas y rotación de bits.
- Saltos condicionales, ciclos y espera activa.

> Las direcciones corresponden al sistema hardware descrito por el archivo adjunto. El nombre de la placa, por sí solo, no garantiza ese mapa: el sistema configurado en la FPGA debe implementar los periféricos en esas direcciones.

## 2. Relación entre los archivos

El programa comienza con:

```assembly
.include "address_map_nios2.s"
```

La directiva `.include` incorpora el contenido del archivo indicado durante el ensamblado. Así, el programa puede utilizar nombres simbólicos en lugar de escribir directamente las direcciones hexadecimales.

Las constantes utilizadas son:

| Símbolo | Dirección | Uso en el programa |
| --- | --- | --- |
| `LED_BASE` | `0xFF200000` | Escritura del patrón de LEDs |
| `SW_BASE` | `0xFF200040` | Lectura de interruptores |
| `KEY_BASE` | `0xFF200050` | Lectura de pulsadores |

Por ejemplo:

```assembly
.equ SW_BASE, 0xFF200040
```

La directiva `.equ` asigna un valor constante a un símbolo; no reserva una variable en memoria ni configura el hardware.

En el archivo adjunto, `LED_BASE` y `LEDR_BASE` tienen la misma dirección. Por tanto, este programa utiliza el puerto identificado allí como LEDs rojos. `LEDG_BASE`, ubicado en `0xFF200010`, no se utiliza.

## 3. Entrada/salida mapeada en memoria

El procesador accede a los periféricos mediante direcciones de su espacio de memoria. Una lectura del puerto de interruptores obtiene su estado; una escritura en el puerto de LEDs modifica sus salidas.

En este programa:

```assembly
ldwio r4, 0(r15)
stwio r6, 0(r16)
```

- `ldwio` lee una palabra de 32 bits usando un acceso de entrada/salida.
- `stwio` escribe una palabra de 32 bits usando un acceso de entrada/salida.
- `0(r15)` representa la dirección almacenada en `r15`, más un desplazamiento de cero bytes.

Las instrucciones de E/S evitan que una caché de datos sustituya el acceso al periférico por una copia almacenada. El número de bits físicos de un puerto puede ser menor que los 32 bits transferidos por la instrucción.

## 4. Registros utilizados

| Registro | Función |
| --- | --- |
| `r0` | Constante cero del procesador |
| `r4` | Estado de los interruptores; después, valor temporal para construir el patrón |
| `r5` | Estado leído de los pulsadores |
| `r6` | Patrón de 32 bits que se escribe en el puerto de LEDs y se rota |
| `r7` | Contador del retardo |
| `r15` | Dirección base del puerto de interruptores |
| `r16` | Dirección base del puerto de LEDs |
| `r17` | Dirección base del puerto de pulsadores |
| `r18` | Dirección del dato inicial `LED_bits` |

Los registros conservan direcciones o datos según el uso que les dé el programa. Por ejemplo, `r18` contiene una dirección; `r6` recibe el contenido leído en esa dirección.

## 5. Paso 1: declarar el código y el punto de entrada

```assembly
.text
.global _start
_start:
```

- `.text` selecciona la sección de código.
- `.global _start` hace visible el símbolo `_start` para otras etapas de construcción, como el enlazado.
- `_start:` define una etiqueta en la posición actual del código.

Para que la ejecución comience allí, la configuración de construcción y carga debe utilizar ese punto de entrada. La etiqueta no es una instrucción ejecutable.

## 6. Paso 2: cargar las direcciones base

```assembly
movia r15, SW_BASE
movia r16, LED_BASE
movia r17, KEY_BASE
movia r18, LED_bits
```

`movia` es una pseudoinstrucción que permite cargar una dirección completa en un registro. El ensamblador puede expandirla en varias instrucciones máquina.

Después de estas operaciones:

| Registro | Valor |
| --- | --- |
| `r15` | `0xFF200040` |
| `r16` | `0xFF200000` |
| `r17` | `0xFF200050` |
| `r18` | Dirección asignada a `LED_bits` al construir el ejecutable |

Estas instrucciones cargan las direcciones; todavía no leen los interruptores ni los pulsadores.

## 7. Paso 3: obtener el patrón inicial

```assembly
ldwio r6, 0(r18)
```

Al final del archivo aparece:

```assembly
.data
LED_bits:
.word 0x0F0F0F0F
```

- `.data` selecciona la sección de datos.
- `LED_bits:` identifica la dirección del dato.
- `.word` reserva e inicializa una palabra de 32 bits.

La carga deja:

```text
r6 = 0x0F0F0F0F
   = 00001111 00001111 00001111 00001111₂
```

El byte `00001111` se repite cuatro veces.

**Observación sobre el código original:** aquí se utiliza `ldwio` para leer un dato de memoria ordinaria. Normalmente se emplearía `ldw` para `LED_bits`, reservando `ldwio` para los periféricos. Esta explicación conserva la instrucción del archivo entregado.

## 8. Paso 4: leer los interruptores

```assembly
DO_DISPLAY:
    ldwio r4, 0(r15)
```

`DO_DISPLAY` marca el inicio del ciclo principal. La instrucción lee el estado del puerto `SW` y lo coloca en `r4`.

Para los ejemplos siguientes se supone que el valor leído es:

```text
r4 = 0x00000005
```

Su byte inferior es `00000101`: los bits 0 y 2 están en uno.

La lectura ocurre en cada vuelta del ciclo, pero el valor de los interruptores solo se adopta como patrón cuando se detecta un pulsador presionado.

## 9. Paso 5: comprobar los pulsadores

```assembly
ldwio r5, 0(r17)
beq   r5, r0, NO_BUTTON
```

Primero se lee el puerto de pulsadores. Después, `beq` salta a `NO_BUTTON` si ambos registros son iguales.

El programa supone esta interpretación del puerto:

| Valor leído | Interpretación utilizada por el programa |
| --- | --- |
| `r5 = 0` | Ningún pulsador presionado |
| `r5 ≠ 0` | Al menos un pulsador presionado |

Por tanto:

- Si `r5` es cero, se conserva el patrón de `r6` y se pasa a mostrarlo.
- Si `r5` es distinto de cero, se ejecuta el bloque que construye el nuevo patrón.

> Esta es la polaridad que espera el software. Si otro sistema expone los pulsadores con una polaridad distinta, la condición debe adaptarse a ese hardware.

## 10. Paso 6: copiar los interruptores al patrón

```assembly
mov r6, r4
```

La instrucción copia el valor de `r4` en `r6`.

Con el ejemplo anterior:

```text
r4 = 0x00000005
r6 = 0x00000005
```

Esto modifica un registro del procesador. Los LEDs todavía no se actualizan: la escritura al periférico ocurre más adelante con `stwio`.

## 11. Paso 7: repetir el patrón en los cuatro bytes

```assembly
roli r4, r4, 8
or   r6, r6, r4
roli r4, r4, 8
or   r6, r6, r4
roli r4, r4, 8
or   r6, r6, r4
```

### 11.1. Rotación a la izquierda

`roli` rota los 32 bits del registro hacia la izquierda. Los bits que salen por el extremo superior vuelven a entrar por el inferior.

Por ejemplo:

```text
rotación izquierda de 1 bit:
0x80000001 → 0x00000003
```

A diferencia de un desplazamiento lógico, la rotación conserva todos los bits y cambia sus posiciones.

### 11.2. Operación OR

La instrucción:

```assembly
or r6, r6, r4
```

combina los registros bit a bit. Cada bit del resultado es uno si el bit correspondiente de cualquiera de los operandos es uno.

### 11.3. Traza del ejemplo

Si inicialmente `r4 = 0x00000005`, la secuencia produce:

| Operación ejecutada | `r4` | `r6` |
| --- | --- | --- |
| `mov r6, r4` | `0x00000005` | `0x00000005` |
| Primera rotación de 8 bits | `0x00000500` | `0x00000005` |
| Primer OR | `0x00000500` | `0x00000505` |
| Segunda rotación de 8 bits | `0x00050000` | `0x00000505` |
| Segundo OR | `0x00050000` | `0x00050505` |
| Tercera rotación de 8 bits | `0x05000000` | `0x00050505` |
| Tercer OR | `0x05000000` | `0x05050505` |

El resultado es:

```text
r6 = 00000101 00000101 00000101 00000101₂
```

La repetición permite que, al rotar los 32 bits, el byte inferior reciba continuamente la misma secuencia circular de ocho bits.

### 11.4. Precisión sobre los ocho bits

Los comentarios del programa hablan de mostrar únicamente los ocho bits inferiores. Sin embargo, el archivo de direcciones identifica la salida como `LEDR_BASE` y no especifica el ancho del puerto. El código tampoco enmascara la lectura de los interruptores.

Por ello, la interpretación de «repetir un byte» supone que el valor de entrada está limitado a los ocho bits inferiores. Si otros bits de `SW` están activos, también participan en las rotaciones y los OR, y pueden modificar el resultado.

Si se quisiera limitar explícitamente el patrón a ocho interruptores, podría añadirse, antes de la copia:

```assembly
andi r4, r4, 0xFF
mov  r6, r4
```

Este sería un cambio opcional al programa original. Además, la cantidad de LEDs que muestran el dato depende del ancho real del periférico: escribir una palabra de 32 bits no implica que existan 32 LEDs.

## 12. Paso 8: esperar a liberar los pulsadores

```assembly
WAIT:
    ldwio r5, 0(r17)
    bne   r5, r0, WAIT
```

`bne` salta si los registros son diferentes. Mientras `r5` sea distinto de cero, el procesador vuelve a leer los pulsadores.

Cuando todos se liberan, la lectura esperada es cero y el programa continúa.

Este mecanismo es una **espera activa**: el procesador ejecuta instrucciones repetidamente mientras espera un cambio externo.

Durante esta espera:

- El patrón de los LEDs permanece en el último valor escrito al puerto.
- El nuevo patrón ya está preparado en `r6`, pero aún no se ha escrito.
- Los interruptores no se vuelven a leer dentro de `WAIT`.
- Si se cambian los interruptores mientras se mantiene presionado el pulsador, esos cambios no actualizan el patrón ya capturado.

El bloque espera la liberación, pero no implementa un filtrado explícito del rebote mecánico del pulsador.

## 13. Paso 9: escribir el patrón en los LEDs

```assembly
NO_BUTTON:
    stwio r6, 0(r16)
```

La instrucción escribe el contenido de `r6` en la dirección `LED_BASE`.

A esta etiqueta se llega por dos caminos:

1. No se detectó ningún pulsador presionado.
2. Se capturó un nuevo patrón y terminó la espera de liberación.

Aunque la etiqueta se llama `NO_BUTTON`, también recibe la ejecución después de soltar un pulsador.

## 14. Paso 10: preparar la siguiente posición

```assembly
roli r6, r6, 1
```

Después de escribir el puerto, el programa rota `r6` una posición a la izquierda. El nuevo valor queda preparado para la próxima escritura.

**Orden observable:** primero se muestra el patrón actual; después se calcula el siguiente. Modificar `r6` no cambia por sí solo el puerto de LEDs.

Para el patrón inicial, los valores escritos sucesivamente son:

| Escritura | Palabra enviada al puerto | Ocho bits inferiores |
| --- | --- | --- |
| 1 | `0x0F0F0F0F` | `00001111` |
| 2 | `0x1E1E1E1E` | `00011110` |
| 3 | `0x3C3C3C3C` | `00111100` |
| 4 | `0x78787878` | `01111000` |
| 5 | `0xF0F0F0F0` | `11110000` |
| 6 | `0xE1E1E1E1` | `11100001` |
| 7 | `0xC3C3C3C3` | `11000011` |
| 8 | `0x87878787` | `10000111` |
| 9 | `0x0F0F0F0F` | `00001111` |

La tabla presenta los ocho bits inferiores para facilitar el seguimiento; un puerto más ancho puede mostrar bits adicionales de la misma palabra. La dirección física aparente del movimiento depende de cómo estén ordenados los LEDs en la placa.

## 15. Paso 11: generar un retardo

```assembly
movia r7, 1500000
DELAY:
    subi r7, r7, 1
    bne  r7, r0, DELAY
```

El programa carga el contador y lo reduce hasta llegar a cero:

```text
1500000, 1499999, 1499998, …, 2, 1, 0
```

`subi` resta el valor inmediato uno. `bne` repite el ciclo mientras el contador sea distinto de cero.

En cada ejecución completa del retardo se realizan:

- 1 500 000 ejecuciones de `subi`.
- 1 500 000 ejecuciones de `bne`; la última no toma el salto.

El retardo incluye, por tanto, 3 000 000 de ejecuciones de esas dos instrucciones. Esto **no equivale necesariamente a 3 000 000 de ciclos de reloj**.

Una estimación es:

\[
T_{\text{retardo}} \approx \frac{C_{\text{retardo}}}{f_{\text{CPU}}}
\]

El número de ciclos depende de la implementación de Nios II, del comportamiento de los saltos y del acceso a las instrucciones. La constante `1500000` no representa directamente segundos ni milisegundos.

El retardo ocupa al procesador. Además, como no se leen los pulsadores durante `DELAY`, una pulsación breve que ocurra completamente dentro de ese intervalo puede pasar inadvertida.

## 16. Paso 12: repetir el ciclo principal

```assembly
br DO_DISPLAY
```

`br` realiza un salto incondicional al inicio del ciclo. El programa vuelve a leer los interruptores y los pulsadores.

No existe una condición de finalización: continúa hasta que se detiene o reinicia el sistema.

## 17. Directiva final

```assembly
.end
```

Marca el final del código fuente para el ensamblador. No es una instrucción de parada del procesador. En este programa, la ejecución permanece en el ciclo principal.

## 18. Algoritmo equivalente

El siguiente pseudocódigo resume el comportamiento, suponiendo la polaridad de pulsadores utilizada por el programa y operaciones de 32 bits:

```text
patrón ← 0x0F0F0F0F

repetir indefinidamente:
    interruptores ← leer puerto SW
    pulsadores ← leer puerto KEY

    si pulsadores ≠ 0:
        temporal ← interruptores
        patrón ← temporal

        repetir 3 veces:
            temporal ← rotar temporal 8 bits a la izquierda
            patrón ← patrón OR temporal

        repetir:
            pulsadores ← leer puerto KEY
        hasta que pulsadores = 0

    escribir patrón en el puerto LED
    patrón ← rotar patrón 1 bit a la izquierda

    contador ← 1500000
    repetir:
        contador ← contador - 1
    hasta que contador = 0
```

## 19. Resumen de instrucciones y directivas

| Elemento | Función en este programa |
| --- | --- |
| `.include` | Incorpora las definiciones del mapa de direcciones |
| `.equ` | Define constantes simbólicas |
| `.text` | Selecciona la sección de código |
| `.global` | Declara un símbolo visible fuera del archivo objeto |
| `.data` | Selecciona la sección de datos |
| `.word` | Reserva e inicializa una palabra de 32 bits |
| `.end` | Marca el final del fuente para el ensamblador |
| `movia` | Carga una dirección o constante completa mediante una pseudoinstrucción |
| `mov` | Copia un registro |
| `ldwio` | Lee una palabra mediante un acceso de E/S |
| `stwio` | Escribe una palabra mediante un acceso de E/S |
| `roli` | Rota los 32 bits a la izquierda una cantidad inmediata |
| `or` | Combina los operandos mediante OR bit a bit |
| `beq` | Salta si los registros son iguales |
| `bne` | Salta si los registros son distintos |
| `subi` | Resta una constante inmediata |
| `br` | Salta incondicionalmente |

Las directivas orientan el ensamblado; las instrucciones describen operaciones que ejecutará el procesador. Algunas formas del fuente son pseudoinstrucciones y no tienen necesariamente una correspondencia uno a uno con instrucciones máquina.

## 20. Preguntas para comprobar la comprensión

1. ¿Por qué `movia r15, SW_BASE` no lee el estado de los interruptores?
2. ¿Qué diferencia existe entre el contenido de `r18` y el de `r6` después de cargar el patrón inicial?
3. ¿Cuándo se copia el valor de los interruptores al patrón?
4. ¿Por qué se repite el byte en cuatro posiciones antes de rotar?
5. ¿Qué ocurre si se mantiene un pulsador presionado?
6. ¿Qué valor se utiliza si los interruptores cambian durante `WAIT`?
7. ¿Por qué el retardo puede perder una pulsación breve?
8. ¿Qué cambia al reducir el contador de `1500000` a `750000`?
9. ¿Por qué 3 000 000 de instrucciones no implican necesariamente 3 000 000 de ciclos?
10. ¿Qué diferencia habría entre rotar y desplazar el patrón hacia la izquierda?

## 21. Aspectos que deben verificarse al ejecutar el ejemplo

- El hardware cargado debe corresponder al mapa de direcciones utilizado.
- La polaridad de los pulsadores debe coincidir con las condiciones del programa.
- El ancho de los puertos determina cuántos interruptores y LEDs intervienen físicamente.
- Para interpretar la construcción como repetición de un byte, deben utilizarse solo los ocho bits inferiores de entrada o aplicar una máscara.
- La velocidad de la animación depende del procesador y del sistema de memoria.

**Base de esta explicación:** análisis de los dos archivos suministrados. Las trazas de bits se calcularon a partir de las operaciones del programa; no se realizó una ejecución sobre la placa.
