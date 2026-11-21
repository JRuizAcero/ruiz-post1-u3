# ruiz-post1-u3

# Laboratorio U3 — Manejo del DEBUG: Exploración, Ensamblado y Ejecución Paso a Paso en DOSBox

**Curso:** Arquitectura de Computadores · Unidad 3 · Post-contenido
**Estudiante:** Jesús Enrique Ruiz Acero
**Repositorio:** `ruiz-post1-u3`

---

## Descripción general

Este repositorio documenta un laboratorio guiado de dos partes ejecutado en **DOSBox 0.74-3** con el depurador **DEBUG** (versión FreeDOS, Debug/X 2.51).

- **Parte 1 — Exploración con DEBUG:** el estudiante inspeccionó y modificó los registros con `R`. Rellenó, modificó y volcó memoria con `F`, `E` y `D`. Desensambló con `U`, ensambló con `A` y leyó un valor de memoria con direccionamiento directo (`MOV AX,[0300]`).
- **Parte 2 — Ensamblado y ejecución paso a paso:** el estudiante ensambló tres programas. Los ejecutó instrucción a instrucción con `T`, registró su evolución en tablas de traza y comparó el bucle `LOOP` con su equivalente `DEC`/`JNZ`. Por último, usó `G` como alternativa de verificación rápida.

### Observaciones sobre el entorno

1. **DEBUG no viene incluido en DOSBox.** El estudiante descargó el `DEBUG.COM` de FreeDOS (Debug/X) y lo copió en la raíz de la carpeta montada `C:\DOSWork`. También agregó `PATH Z:\;C:\` al `[autoexec]` de DOSBox para poder invocarlo desde cualquier subcarpeta. Una primera versión de DEBUG obtenida de otra fuente no reconocía el comando `A` (respondía `^ Error` sobre la letra del comando). Por eso se reemplazó por el DEBUG de FreeDOS, que implementa todos los comandos del laboratorio.
2. **Límite de 8 caracteres en los nombres de carpeta.** Los nombres `LAB3POST1` y `LAB3POST2` de la guía tienen 9 caracteres y exceden el formato 8.3 de DOS. DOSBox truncó `LAB3POST1` a `LAB3POST`, y por eso el prompt muestra `C:\LAB3POST>`. Luego rechazó `MD LAB3POST2` con el mensaje `Unable to make: LAB3POST2.`, porque el nombre truncado coincidía con la carpeta existente. Se usaron las carpetas `LAB3POST` (Parte 1) y `LAB3POS2` (Parte 2).
3. **Segmento de trabajo.** El segmento asignado por DOS en esta máquina fue `0725`, distinto del `1357` de la guía. Es esperable: depende de la memoria ocupada antes de cargar DEBUG. Todas las direcciones de este documento usan el segmento real observado.

---

# PARTE 1 — Exploración con DEBUG en DOSBox

## Comandos utilizados

| Comando | Función | Uso en el laboratorio |
|---|---|---|
| `R` | Muestra los registros y banderas | Estado inicial del procesador |
| `R AX` | Modifica un registro | Cargar 0x1234 en AX |
| `F rango patrón` | Rellena memoria con un patrón cíclico | `F 200 L40 AB CD EF`, `F 300 L10 00` |
| `D rango` | Vuelca memoria en hexadecimal y ASCII | `D 200 L40`, `D 0 L20`, `D 300 L10` |
| `E dir bytes` | Escribe bytes puntuales | `E 300 78 56` |
| `U rango` | Desensambla | `U 100 L10`, `U 100 109`, `U 320 324` |
| `A dir` | Ensambla instrucciones en memoria | `A 100`, `A 320` |
| `T` | Ejecuta una instrucción | Verificar `MOV AX,[0300]` |

## Parte A — Configuración del entorno

```
Z:\> MOUNT C C:\DOSWork
Drive C is mounted as local directory C:\DOSWork\
Z:\> C:
C:\> MD LAB3POST1
C:\> CD LAB3POST1
C:\LAB3POST> DEBUG
-
```

## Parte B — Registros (Checkpoint 1)

```
-R
-R AX
AX 0000
:1234
-R
```

Con el primer `R`, AX, BX, CX y DX aparecen en cero. Los cuatro registros de segmento (DS, ES, SS, CS) tienen el mismo valor, porque un programa .COM comparte un único segmento con el PSP. IP vale `0100`, la primera dirección ejecutable después de los 256 bytes del PSP. SP indica el tope inicial de la pila.

Después de `R AX` con el valor `1234`, el siguiente `R` muestra `AX=1234` (4660 decimal) y todos los demás registros sin cambios. Esto confirma que la modificación es selectiva.

## Parte C — Volcado y relleno de memoria (Checkpoint 2)

```
-F 200 L40 AB CD EF
-D 200 L40
```

`F` no muestra mensaje de confirmación. El retorno al prompt `-` indica que se ejecutó sin errores. El volcado muestra el patrón `AB CD EF` repetido cíclicamente en las 4 filas de 16 bytes (64 bytes = 40h).

**Columnas de la salida del comando D:**
- La **primera columna** es la dirección `segmento:desplazamiento` del primer byte de cada fila.
- La **columna central** muestra 16 bytes en hexadecimal. El guion separa los primeros 8 de los últimos 8 para facilitar la lectura.
- La **columna derecha** muestra esos mismos 16 bytes como caracteres ASCII. DEBUG representa con un punto los bytes que no son imprimibles (fuera del rango 20h–7Eh). Por eso los valores ABh, CDh y EFh no producen caracteres legibles.
- Cada fila avanza 10h (16) posiciones respecto a la anterior: 0200, 0210, 0220 y 0230.

**PSP (`D 0 L20`):** los dos primeros bytes del segmento son `CD 20`, que corresponden a la instrucción `INT 20`. El DOS coloca esta instrucción al inicio del PSP como mecanismo de terminación: un programa .COM que ejecute `RET` con la pila inicial salta a la dirección 0000 y termina de forma controlada.

## Parte D — Desensamblado y ensamblado (Checkpoint 3)

**Paso 9 — `U 100 L10` sin programa cargado:** DEBUG desensambló como instrucciones los bytes residuales que había en memoria. Demuestra que en modo real no existe distinción entre código y datos: el procesador interpreta como instrucción cualquier byte al que apunte CS:IP.

**Paso 10 — Ensamblado con A y verificación con U:**

```
-A 100
0725:0100 MOV AX, 0005
0725:0103 MOV BX, 0003
0725:0106 ADD AX,BX
0725:0108 INT 20
0725:010A
-U 100 109
0725:0100 B80500        MOV     AX,0005
0725:0103 BB0300        MOV     BX,0003
0725:0106 01D8          ADD     AX,BX
0725:0108 CD20          INT     20
```

| Instrucción | Bytes | Explicación |
|---|---|---|
| `MOV AX,0005` | `B8 05 00` | Opcode B8 (MOV AX,imm16) + inmediato 0005 en little-endian |
| `MOV BX,0003` | `BB 03 00` | Opcode BB (MOV BX,imm16) + inmediato 0003 en little-endian |
| `ADD AX,BX` | `01 D8` | Opcode 01 (ADD r/m16,r16) + byte ModR/M D8 |
| `INT 20` | `CD 20` | Opcode CD (INT) + número de interrupción 20h |

El programa ocupa 10 bytes (0100–0109).

**Observación sobre `ADD AX,BX`:** la guía muestra la codificación `03 C3` y el DEBUG de FreeDOS produjo `01 D8`. Ambas son codificaciones válidas de la misma instrucción. El 8086 tiene dos opcodes de ADD entre registros: `03 /r` (ADD r16, r/m16) y `01 /r` (ADD r/m16, r16). En `03 C3` el destino AX va en el campo *reg* y BX en el campo *r/m*. En `01 D8` ocurre lo contrario: BX en *reg* y AX en *r/m*. Las dos ocupan 2 bytes y producen el mismo resultado. Cada ensamblador elige una de las dos formas.

## Parte E — Modificación con E y direccionamiento directo (Checkpoint 4)

```
-F 300 L10 00
-D 300 L10
0725:0300  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00 ................
-E 300 78 56
-D 300 L10
0725:0300  78 56 00 00 00 00 00 00-00 00 00 00 00 00 00 00 xV..............
```

El segundo volcado confirma que `E` modificó solo los dos primeros bytes (78h en 0300 y 56h en 0301). Los 14 restantes siguen en 00h, como los dejó `F`. Leídos como palabra en formato little-endian (byte menos significativo en la dirección menor), estos dos bytes representan **0x5678**. La columna ASCII muestra `xV` porque 78h es la `x` y 56h es la `V`.

```
-A 320
0725:0320 MOV AX, [0300]
0725:0323 INT 20
0725:0325
-U 320 324
0725:0320 A10003        MOV     AX,[0300]
0725:0323 CD20          INT     20
-R IP
IP 0100
:320
-T
AX=5678 BX=0000 CX=0000 DX=0000 SP=FFFE BP=0000 SI=0000 DI=0000
DS=0725 ES=0725 SS=0725 CS=0725 IP=0323 NV UP EI PL NZ NA PO NC
0725:0323 CD20          INT     20
```

Al ejecutar `MOV AX,[0300]` con `T`, **AX tomó el valor 5678**, el que se escribió con `E`. IP avanzó 3 bytes, a 0323. El programa se ensambló en 0320 para no sobrescribir el del Paso 10 (0100–0109).

### Decisión técnica 1 — Verificación no destructiva de una escritura en memoria (Paso 12)

Para confirmar que `E 300 78 56` tuvo el efecto esperado, el comando correcto es **D**. De los cuatro comandos disponibles es el único verdaderamente de solo lectura.

Invocar `E 300` sin lista de bytes no es una opción segura. En ese modo, DEBUG entra en edición interactiva: muestra el valor actual de cada byte y espera una tecla. Cualquier valor tecleado por error sobrescribe el byte que se quería verificar, y la verificación termina alterando el dato. En cambio, `E 300 78 56` con la lista completa es una escritura explícita y sin interacción: hace exactamente lo indicado.

**F** tampoco sirve. Aunque opera sobre un rango, igual que D, su única función es escribir un patrón en todo el rango. Usarlo para verificar destruiría precisamente los datos que se pretende comprobar.

**R** solo lee y modifica registros, no memoria, así que no muestra el contenido de DS:0300. Además, `R` con un nombre de registro también abre un modo de modificación.

**D** solo lee la memoria y la presenta en hexadecimal y ASCII. No tiene ningún modo en que escriba, por lo que su ejecución no puede alterar el estado del programa. Una verificación debe observar sin modificar lo observado, y solo D cumple esa condición.

### Decisión técnica 2 — Direccionamiento inmediato vs. directo a memoria (Paso 14)

`MOV AX,0005` (`B8 05 00`) y `MOV AX,[0300]` (`A1 00 03`) ocupan los mismos 3 bytes, pero no cuestan lo mismo al ejecutarse.

En el modo **inmediato**, el operando fuente (0005) viaja embebido en el flujo de instrucciones. El procesador lo obtiene en la misma búsqueda (fetch) que el opcode y no necesita ningún acceso adicional a memoria.

En el modo **directo a memoria**, los dos bytes que siguen al opcode no son el dato sino la dirección donde está el dato. Después de leer la instrucción, el procesador debe calcular la dirección efectiva DS:0300 y realizar **un ciclo adicional de lectura en el bus de memoria** para traer la palabra almacenada ahí. Por eso el directo es más lento que el inmediato.

El estudiante preferiría el direccionamiento directo cuando el objetivo no es la velocidad sino leer un valor que puede cambiar en tiempo de ejecución. Es el caso del 5678 escrito con `E` en el Paso 11. Con direccionamiento inmediato, el valor queda fijo al momento de ensamblar. Con direccionamiento directo, la instrucción lee lo que haya en esa posición cuando se ejecuta, sea cual sea el valor.

Para saber qué codificación produjo cada comando `A` sin ejecutar ninguna instrucción, basta usar **U**. El opcode `B8` identifica MOV AX con valor inmediato. El opcode `A1` identifica MOV AX con dirección directa de memoria, y `U` además muestra el operando entre corchetes: `[0300]`.

---

# PARTE 2 — Ensamblado y ejecución paso a paso

## Comandos utilizados

| Comando | Función |
|---|---|
| `A 100`, `A 200` | Ensamblar los programas |
| `U` | Verificar la codificación antes de ejecutar |
| `R IP` | Posicionar el puntero de instrucción al inicio del programa |
| `T` | Ejecutar una instrucción y mostrar el estado (traza) |
| `P` | Ejecutar `INT 20` como un solo paso |
| `D CS:100 L0D` | Ver los bytes del programa con bucle |
| `G 20C` | Ejecutar a velocidad completa hasta un punto de parada |

**Observación sobre `INT 20` y el comando T:** en el DEBUG real, `T` sobre una instrucción `INT` no ejecuta la interrupción como un solo paso: entra en la rutina de servicio del sistema y traza sus instrucciones internas. Para ejecutar `INT 20` completa y obtener `Program terminated normally`, se usó el comando **P** (*proceed*). En los conteos de instrucciones ejecutadas, esa ejecución cuenta como un paso, igual que en la guía.

## Parte A — Programa de suma (Checkpoint 1)

```
C:\> MD LAB3POS2
C:\> CD LAB3POS2
C:\LAB3POS2> DEBUG
-A 100
0725:0100 MOV AX, 000A
0725:0103 MOV BX, 0005
0725:0106 MOV CX, 0003
0725:0109 ADD AX,BX
0725:010B ADD AX,CX
0725:010D INT 20
0725:010F
-U 100 10E
0725:0100 B80A00        MOV     AX,000A
0725:0103 BB0500        MOV     BX,0005
0725:0106 B90300        MOV     CX,0003
0725:0109 01D8          ADD     AX,BX
0725:010B 01C8          ADD     AX,CX
0725:010D CD20          INT     20
-R IP
IP 0100
:100
-T  (×5)
```

### Tabla de traza — programa de suma

| # | Instrucción ejecutada | AX | BX | CX | IP siguiente | ZF | CF | SF |
|---|---|---|---|---|---|---|---|---|
| 1 | `MOV AX,000A` | 000A | 0000 | 0000 | 0103 | 0 (NZ) | 0 (NC) | 0 (PL) |
| 2 | `MOV BX,0005` | 000A | 0005 | 0000 | 0106 | 0 (NZ) | 0 (NC) | 0 (PL) |
| 3 | `MOV CX,0003` | 000A | 0005 | 0003 | 0109 | 0 (NZ) | 0 (NC) | 0 (PL) |
| 4 | `ADD AX,BX` | 000F | 0005 | 0003 | 010B | 0 (NZ) | 0 (NC) | 0 (PL) |
| 5 | `ADD AX,CX` | 0012 | 0005 | 0003 | 010D | 0 (NZ) | 0 (NC) | 0 (PL) |

**Resultado:** AX = **0x0012 = 18 decimal** (10 + 5 + 3).

**Observaciones:**
- Las instrucciones `MOV` no modifican las banderas.
- Las dos `ADD` dejan ZF = 0 porque el resultado no es cero, CF = 0 porque no hubo acarreo fuera de los 16 bits, y SF = 0 porque el bit más significativo del resultado es 0 (resultado positivo).
- Como en la Parte 1, DEBUG codificó las sumas entre registros con el opcode `01`: `01 D8` para ADD AX,BX y `01 C8` para ADD AX,CX. Son formas equivalentes a las `03 C3` y `03 C1` de la guía.

## Parte B — Programa con bucle LOOP (Checkpoint 2)

```
-A 100
0725:0100 MOV AX, 0000
0725:0103 MOV CX, 0004
0725:0106 ADD AX, 0002
0725:0109 LOOP 0106
0725:010B INT 20
0725:010D
-U 100 10C
0725:0100 B80000        MOV     AX,0000
0725:0103 B90400        MOV     CX,0004
0725:0106 83C002        ADD     AX,+02
0725:0109 E2FB          LOOP    0106
0725:010B CD20          INT     20
```

`LOOP 0106` se codifica como `E2 FB`. `E2` es el opcode de LOOP y `FB` es el desplazamiento relativo con signo, calculado desde la dirección de la instrucción siguiente: 0106 − 010B = −5 = FBh en complemento a 2 de 8 bits. Los saltos cortos del 8086 alcanzan un rango de −128 a +127 bytes.

Nota: en la guía, `LOOP` aparece en la dirección 010A al ensamblar. En realidad `ADD AX,0002` ocupa 3 bytes (0106–0108), así que `LOOP` queda en **0109** e `INT 20` en **010B**, como confirma `U`.

```
-R IP
:100
-T  (×10)
-P
```

### Tabla de traza — bucle LOOP

| Iteración | Instrucción | AX después | CX después | IP siguiente | ¿LOOP salta? |
|---|---|---|---|---|---|
| Inicio | `MOV AX,0000` | 0000 | 0000 | 0103 | — |
| Inicio | `MOV CX,0004` | 0000 | 0004 | 0106 | — |
| 1 | `ADD AX,0002` | 0002 | 0004 | 0109 | — |
| 1 | `LOOP 0106` | 0002 | 0003 | 0106 | Sí |
| 2 | `ADD AX,0002` | 0004 | 0003 | 0109 | — |
| 2 | `LOOP 0106` | 0004 | 0002 | 0106 | Sí |
| 3 | `ADD AX,0002` | 0006 | 0002 | 0109 | — |
| 3 | `LOOP 0106` | 0006 | 0001 | 0106 | Sí |
| 4 | `ADD AX,0002` | 0008 | 0001 | 0109 | — |
| 4 | `LOOP 0106` | 0008 | 0000 | 010B | **No** (CX = 0) |
| Fin | `INT 20` (con P) | — | — | — | Program terminated normally |

**Resultado:** AX = **0x0008 = 8 decimal** (2 × 4) cuando LOOP no salta porque CX llegó a 0000. En cada iteración, LOOP primero decrementa CX y después evalúa si es distinto de cero.

## Parte C — Análisis del código máquina (Paso 8)

```
-D CS:100 L0D
0725:0100  B8 00 00 B9 04 00 83 C0-02 E2 FB CD 20
```

| Bytes | Instrucción | Tamaño |
|---|---|---|
| `B8 00 00` | MOV AX,0000 | 3 bytes |
| `B9 04 00` | MOV CX,0004 | 3 bytes |
| `83 C0 02` | ADD AX,+02 | 3 bytes |
| `E2 FB` | LOOP 0106 | 2 bytes |
| `CD 20` | INT 20 | 2 bytes |
| | **Total** | **13 bytes** |

**Observaciones:**
- La guía usa `D CS:100 L0C`, que muestra solo 12 bytes y deja fuera el `20` final de `INT 20`. El programa ocupa 13 bytes (0100–010C), así que la longitud correcta es `L0D`.
- DEBUG codificó `ADD AX,0002` como `83 C0 02` (ADD r/m16 con inmediato de 8 bits extendido en signo), no como `05 02 00` (ADD AX con inmediato de 16 bits), que es lo que muestra la guía. Ambas ocupan 3 bytes, así que las direcciones del programa no cambian.

## Parte D — Bucle equivalente con DEC/JNZ (Checkpoint 3)

```
-A 200
0725:0200 MOV AX, 0000
0725:0203 MOV CX, 0004
0725:0206 ADD AX, 0002
0725:0209 DEC CX
0725:020A JNZ 0206
0725:020C INT 20
0725:020E
-U 200 20D
0725:0200 B80000        MOV     AX,0000
0725:0203 B90400        MOV     CX,0004
0725:0206 83C002        ADD     AX,+02
0725:0209 49            DEC     CX
0725:020A 75FA          JNZ     0206
0725:020C CD20          INT     20
```

`DEC CX` ocupa 1 byte (`49`) y `JNZ 0206` ocupa 2 (`75 FA`). El desplazamiento se calcula igual que en LOOP: 0206 − 020C = −6 = FAh. El mecanismo de control usa 3 bytes en dos instrucciones, frente a los 2 bytes en una sola instrucción de `LOOP` (`E2 FB`).

```
-R IP
IP 0100
:200
-T  (×14)
-P
```

### Tabla de traza — bucle DEC/JNZ

| Iteración | Instrucción | AX después | CX después | IP siguiente | ¿JNZ salta? |
|---|---|---|---|---|---|
| Inicio | `MOV AX,0000` | 0000 | 0000 | 0203 | — |
| Inicio | `MOV CX,0004` | 0000 | 0004 | 0206 | — |
| 1 | `ADD AX,0002` | 0002 | 0004 | 0209 | — |
| 1 | `DEC CX` | 0002 | 0003 | 020A | — (ZF = 0) |
| 1 | `JNZ 0206` | 0002 | 0003 | 0206 | Sí |
| 2 | `ADD AX,0002` | 0004 | 0003 | 0209 | — |
| 2 | `DEC CX` | 0004 | 0002 | 020A | — (ZF = 0) |
| 2 | `JNZ 0206` | 0004 | 0002 | 0206 | Sí |
| 3 | `ADD AX,0002` | 0006 | 0002 | 0209 | — |
| 3 | `DEC CX` | 0006 | 0001 | 020A | — (ZF = 0) |
| 3 | `JNZ 0206` | 0006 | 0001 | 0206 | Sí |
| 4 | `ADD AX,0002` | 0008 | 0001 | 0209 | — |
| 4 | `DEC CX` | 0008 | 0000 | 020A | — (**ZF = 1**, aparece `ZR`) |
| 4 | `JNZ 0206` | 0008 | 0000 | 020C | **No** |
| Fin | `INT 20` (con P) | — | — | — | Program terminated normally (0000) |

**Resultado:** AX = **0x0008**, igual que con `LOOP`. Los dos mecanismos son semánticamente equivalentes.

En la traza se observa que `DEC CX` actualiza ZF: muestra `NZ` mientras CX ≠ 0 y cambia a `ZR` cuando CX llega a 0000. `JNZ` solo consulta esa bandera. `LOOP`, en cambio, no modifica ninguna bandera: decrementa CX y lo compara internamente.

### Comparación de instrucciones ejecutadas

| Versión | Inicialización | Instrucciones por iteración | Iteraciones | Terminación | **Total ejecutado** | Bytes de control |
|---|---|---|---|---|---|---|
| `LOOP` | 2 | 2 (ADD + LOOP) | 4 | 1 | **11** | 2 (`E2 FB`) |
| `DEC`/`JNZ` | 2 | 3 (ADD + DEC + JNZ) | 4 | 1 | **15** | 3 (`49 75 FA`) |

La versión DEC/JNZ ejecuta **4 instrucciones más**, exactamente una por cada iteración: la instrucción de control adicional que necesita frente a LOOP. El conteo es coherente con los bytes de código: un byte más de control por iteración se traduce en una instrucción más extraída y ejecutada en cada vuelta.

### Verificación rápida con G

```
-R IP
IP 0100
:200
-G 20C
AX=0008 BX=0000 CX=0000 DX=0000 SP=FFFE BP=0000 SI=0000 DI=0000
DS=0725 ES=0725 SS=0725 CS=0725 IP=020C NV UP EI PL ZR NA PE NC
0725:020C CD20          INT     20
```

`G 20C` ejecutó el programa completo y se detuvo justo antes de `INT 20`, con **AX = 0008** y CX = 0000. El resultado coincide con la traza hecha con T.

### Decisión técnica 3 — LOOP vs. DEC/JNZ (Paso 10)

Para un bucle contador simple como este, en el que CX no se usa para nada más dentro del cuerpo, el estudiante recomienda **LOOP**.

En bytes de código, LOOP resuelve el control del bucle con 2 bytes (`E2 FB`) en una sola instrucción. DEC/JNZ necesita 3 bytes (`49 75 FA`) repartidos en dos instrucciones. En ejecución, LOOP obliga al procesador a extraer (fetch) y decodificar una sola instrucción de control por iteración, mientras DEC/JNZ exige dos. La traza lo confirma: 11 instrucciones ejecutadas con LOOP frente a 15 con DEC/JNZ para las mismas 4 iteraciones. Con más iteraciones la diferencia crece de forma lineal.

DEC/JNZ es preferible cuando LOOP no alcanza a expresar el bucle. Un caso es que el cuerpo del bucle necesite usar CX para otra operación, por ejemplo como contador de un desplazamiento (`SHL AX,CL`) o en una instrucción de cadena con `REP`. LOOP está atado a CX, mientras DEC/JNZ puede usar cualquier registro como contador (`DEC SI`, `DEC BX`). Otro caso es que la condición de salida no sea simplemente "CX ≠ 0" sino el resultado de una comparación, por ejemplo `CMP AX,0010` seguido de `JNE`. En esa situación se requiere un salto condicional sobre las banderas, que LOOP no evalúa.

Para verificar con U, sin ejecutar nada, cuál versión ocupa menos bytes, basta comparar las direcciones del listado. LOOP ocupa 0109–010A (2 bytes) y DEC + JNZ ocupan 0209–020B (3 bytes). Los bytes de cada instrucción también aparecen en la segunda columna de U.

### Decisión técnica 4 — Comando de verificación: T vs. G (Paso 12)

Si el contador inicial fuera CX = 0064h (100 iteraciones), verificar el resultado con T exigiría más de 300 invocaciones (3 instrucciones por iteración en la versión DEC/JNZ, más la inicialización). Además de lento, eso favorece errores al transcribir. `G 20C` ejecuta el programa a velocidad completa y se detiene en la dirección indicada, justo antes de `INT 20`. Con una sola orden muestra el estado final y confirma AX.

Lo que se pierde con G es toda la información intermedia: los valores de AX y CX en cada vuelta, el cambio de ZF de `NZ` a `ZR`, la dirección a la que salta cada `JNZ` y el momento exacto en que el bucle deja de saltar. G solo muestra el estado en el punto de parada.

En este laboratorio, T fue imprescindible en los **Pasos 4, 7 y 11** de la Parte 2. Esas tareas consistían en llenar tablas de traza instrucción por instrucción, con el valor de los registros, las banderas y el IP siguiente después de cada una. Ese nivel de detalle solo lo da T, que se detiene después de cada instrucción y muestra el estado completo del procesador. G habría entregado únicamente la última fila de cada tabla.

Después de `G 20C`, para confirmar únicamente el valor final de AX sin ver ningún paso intermedio, el estudiante usaría **`R AX`**. El comando muestra `AX 0008` y espera un valor nuevo; se presiona Enter sin escribir nada para dejarlo sin cambios. Otra opción es `R` sin argumentos, que ya muestra el estado completo en el punto de parada.
7. T es la herramienta para el análisis detallado y G para la verificación rápida. Se complementan.
8. Hubo diferencias con la guía, todas explicadas en este documento: DEBUG no viene incluido en DOSBox, los nombres de carpeta exceden 8 caracteres, `T` sobre `INT 20` entra a la rutina de interrupción, algunas codificaciones son distintas y la longitud del volcado del Paso 8 debe ser `L0D`.
