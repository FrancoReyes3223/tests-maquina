# Testeo de instrucciones RTM32

Acá voy anotando los casos que fui probando en la máquina, con lo que hice y lo que me dio.

---

# Caso 1: ADD

## Descripción
ADD suma dos registros y deja el resultado en un tercero (R[rd] = R[rs] + R[rt]). Quiero ver
que la suma dé bien, que no toque los operandos, y probar un caso con acarreo (0xA + 0xFF)
para ver que el desborde de un byte se propague.

## Instructions
- ADD r3, r1, r2  → r3 = r1 + r2

ADD es tipo R con func 011100. Para armar la palabra de 32 bits fui completando cada campo:
opcode 00000, rs=r1 (00001), rt=r2 (00010), rd=r3 (00011), aux y X en cero, y func 011100.
Pegando todo y pasándolo a hexa me quedó 0x0044301C.

## Precondiciones
- Cargo los dos valores a sumar en los registros: `set r1 0x7` y `set r2 0x3`.
- Pongo la instrucción en la posición 0x0000, que es donde arranca la CPU: `set [0x0000] 0x0044301C`.
- (Para repetir la prueba con otros valores tuve que volver el PC a 0x0000 con `set pc 0x0`,
  ver la nota del final.)

## Code
```
set r1 0x7
set r2 0x3
set [0x0000] 0x0044301C
examine 0x0000
n 1
registers
```

Después dejé la misma instrucción cargada y fui cambiando los registros para probar más casos:
7+3, 7+10 (0xA) y 0xA+0xFF, rebobinando el PC con `set pc 0x0` antes de cada `n 1`.

## Postcondiciones
Con `examine 0x0000` me aseguro de que la instrucción quedó cargada (me devuelve 0x0044301C).
El resultado de la suma queda en el registro r3, así que lo miro con `registers`.

Lo que fui obteniendo:
- 7 + 3   → r3 = 0x0000000A (10)
- 7 + 10  → r3 = 0x00000011 (17)
- 0xA + 0xFF → r3 = 0x00000109 (265)

En los tres casos r1 y r2 quedaron como estaban, el PC pasó de 0x0000 a 0x0004 y CAUSE quedó en 0.

## Conclusiones
Anduvo. La suma dio bien en todos los casos, incluido el de acarreo (0xA + 0xFF = 0x109). No
tocó los operandos ni tiró ninguna excepción.

Nota de método: para repetir la prueba con otros valores hay que rebobinar el PC con
`set pc 0x0` antes de cada `n 1`. Si no, el PC ya quedó en 0x0004 tras la primera ejecución y
`n 1` ejecuta memoria vacía en vez del ADD, y r3 no cambia.

---

# Caso 2: SUB

## Descripción
SUB resta dos registros (R[rd] = R[rs] - R[rt]). Como es casi igual a ADD, la uso para seguir
practicando la codificación y ver qué pasa cuando el resultado da negativo (complemento a dos).

## Instructions
- SUB r3, r1, r2  → r3 = r1 - r2

SUB también es tipo R y lo único que cambia respecto de ADD es el func: acá es 011101 (en ADD
era 011100, o sea cambia solo el último bit). Manteniendo los mismos registros que en el Caso 1,
la palabra me queda igual pero con el último dígito cambiado: 0x0044301D (la C pasa a D).

## Precondiciones
- Cargo los operandos: `set r1` y `set r2`.
- Cargo la instrucción en 0x0000: `set [0x0000] 0x0044301D`.
- Igual que antes, para la segunda prueba rebobino con `set pc 0x0`.

## Code
```
set r1 0x7
set r2 0x3
set [0x0000] 0x0044301D
examine 0x0000
n 1
registers

set r1 0x3
set r2 0x7
set pc 0x0
n 1
registers
```

## Postcondiciones
Reviso con `examine 0x0000` que la instrucción quedó cargada (0x0044301D) y con `registers`
el resultado en r3.

Lo que obtuve:
- 7 - 3 → r3 = 0x00000004 (4)
- 3 - 7 → r3 = 0xFFFFFFFC

## Conclusiones
Anduvo. La resta 7-3 dio 4. En 3-7 el resultado salió 0xFFFFFFFC, que es el -4 en complemento
a dos. La instrucción resta bien y respeta el signo. Los operandos no se modificaron y CAUSE
quedó en 0.

---

# Caso 3: ADDI

## Descripción
ADDI suma un registro con una constante inmediata incluida en la propia instrucción
(R[rt] = R[rs] + imm). Es la primera del formato I que pruebo, así que además de la suma en sí
me interesa el inmediato: que llegue bien a la cuenta y que se extienda con signo, cosa que
verifico sumando un inmediato negativo.

## Instructions
- ADDI rt, rs, imm  → R[rt] = R[rs] + imm  (el inmediato se extiende con signo)

ADDI es tipo I, con un formato distinto al de ADD/SUB: no tiene campo func ni un tercer
registro, sino un inmediato de 17 bits:

```
[31:27] opcode=00001 | [26:22] rs | [21:17] rt | [16:0] imm (17 bits, complemento a dos)
```

Las palabras que armé:
- ADDI r1, r0, 5 → 00001 00000 00001 00000000000000101 → 0x08020005
- ADDI r1, r2, 5 → 0x08820005
- ADDI r1, r2, -1 (imm = 11111111111111111) → 0x0883FFFF

## Precondiciones
- La primera suma la hago contra r0, que está cableado a 0, así el resultado esperado es
  directamente el inmediato.
- Para las otras dos cargo r2 con 0xA, para sumarle 5 y restarle 1.
- Cargo cada instrucción en 0x0000 y rebobino con `set pc 0x0` antes de cada `n 1`.

## Code
```
set [0x0000] 0x08020005
examine 0x0000
set pc 0x0
n 1
registers          # r1 = 0x5

set r2 0xA
set [0x0000] 0x08820005
set pc 0x0
n 1
registers          # r1 = 0xF

set [0x0000] 0x0883FFFF
set pc 0x0
n 1
registers          # r1 = 0x9
```

## Postcondiciones
Miro r1 con `registers`:
- ADDI r1, r0, 5 → r1 = 0x00000005
- ADDI r1, r2, 5 con r2 = 0xA → r1 = 0x0000000F
- ADDI r1, r2, -1 con r2 = 0xA → r1 = 0x00000009

En las tres los registros fuente quedaron intactos y CAUSE en 0.

## Conclusiones
Anduvo. La suma con inmediato da lo esperado (0+5=5 y 10+5=15), y la extensión de signo
también: el imm de 17 bits todo en unos se leyó como -1 y la máquina hizo 10 - 1 = 9, en vez
de sumar el valor como si fuera positivo. No tiró ninguna excepción.

---

# Caso 4: AND, OR, XOR y NOR

## Descripción
Junto las cuatro operaciones lógicas bit a bit en un solo caso, ya que son todas tipo R y
comparten forma (R[rd] = R[rs] op R[rt]). Cargo dos valores con bits conocidos y comparo cada
resultado con la cuenta a mano.

## Instructions
- AND r3, r1, r2 → r3 = r1 & r2
- OR  r3, r1, r2 → r3 = r1 | r2
- XOR r3, r1, r2 → r3 = r1 ^ r2
- NOR r3, r1, r2 → r3 = ~(r1 | r2)

Las cuatro son tipo R con los mismos registros que venía usando (rd=r3, rs=r1, rt=r2), así que
la palabra es igual salvo el func (los últimos 6 bits). Me quedaron:
- AND → func 001000 → 0x00443008
- OR  → func 001001 → 0x00443009
- XOR → func 001010 → 0x0044300A
- NOR → func 001011 → 0x0044300B

## Precondiciones
- Cargo los dos operandos una sola vez: `set r1 0xC` y `set r2 0xA`.
- Los elegí por sus bits: 0xC = 1100 y 0xA = 1010, así se ve claro qué hace cada operación.
- Voy cambiando la instrucción en 0x0000 y rebobino con `set pc 0x0` antes de cada `n 1`.

## Code
```
set r1 0xC
set r2 0xA

set [0x0000] 0x00443008
set pc 0x0
n 1
registers

set [0x0000] 0x00443009
set pc 0x0
n 1
registers

set [0x0000] 0x0044300A
set pc 0x0
n 1
registers

set [0x0000] 0x0044300B
set pc 0x0
n 1
registers
```

## Postcondiciones
Miro r3 con `registers` después de cada una. Comparando con la cuenta a mano (1100 y 1010):
- AND: 1100 & 1010 = 1000 → r3 = 0x00000008
- OR:  1100 | 1010 = 1110 → r3 = 0x0000000E
- XOR: 1100 ^ 1010 = 0110 → r3 = 0x00000006
- NOR: ~(1110) = ...11110001 → r3 = 0xFFFFFFF1

En las cuatro r1 y r2 quedaron intactos y CAUSE en 0.

## Conclusiones
Anduvieron las cuatro y dieron lo que da la cuenta bit a bit. En el NOR el resultado queda
0xFFFFFFF1 porque invierte los 32 bits, no solo los últimos 4. Ninguna tiró excepción.

---

# Caso 5: SLL, SRL y SRA (desplazamientos)

## Descripción
Los tres desplazamientos corren la palabra una cantidad fija de bits: SLL a la izquierda, SRL
a la derecha lógico y SRA a la derecha aritmético. Me interesa la diferencia de relleno entre
SRL y SRA usando un número negativo.

## Instructions
- SLL r3, r1, n → r3 = r1 << n (izquierda)
- SRL r3, r1, n → r3 = r1 >> n lógico (rellena con 0)
- SRA r3, r1, n → r3 = r1 >> n aritmético (rellena con el bit de signo)

Los tres son tipo R, pero a diferencia de ADD o las lógicas no usan un tercer registro: la
cantidad de bits a desplazar es una constante de 5 bits que va en el campo aux (bits [11:7]).
Uso rt=r1 (valor), rd=r3 (resultado) y aux=4 (desplazo 4 bits); cambia solo el func:
- SLL → func 000000 → 0x00023200
- SRL → func 000001 → 0x00023201
- SRA → func 000010 → 0x00023202

## Precondiciones
- Cargo el valor a desplazar en r1.
- Para SLL uso r1 = 0x3 (para ver un corrimiento simple).
- Para SRL y SRA uso r1 = 0xFFFFFFF0 (un negativo), así se nota la diferencia de relleno.
- Rebobino con `set pc 0x0` antes de cada `n 1`.

## Code
```
# SLL: 0x3 << 4
set r1 0x3
set [0x0000] 0x00023200
set pc 0x0
n 1
registers

# SRL: 0xFFFFFFF0 >> 4 logico
set r1 0xFFFFFFF0
set [0x0000] 0x00023201
set pc 0x0
n 1
registers

# SRA: 0xFFFFFFF0 >> 4 aritmetico
set [0x0000] 0x00023202
set pc 0x0
n 1
registers
```

## Postcondiciones
Leo r3 con `registers`:
- SLL con r1=0x3 → r3 = 0x00000030
- SRL con r1=0xFFFFFFF0 → r3 = 0x0FFFFFFF (rellena con ceros)
- SRA con r1=0xFFFFFFF0 → r3 = 0xFFFFFFFF (rellena con el bit de signo, que es 1)

CAUSE quedó en 0 en las tres.

## Diferencia con el manual (SRA)
El manual (Tabla A.2) dice que SLL y SRL operan sobre R[rt], pero que SRA opera sobre R[rs].
Para chequearlo codifiqué SRA poniendo el valor en rs, como indica el manual (palabra
0x00403202, con rs=r1):

```
set r1 0xFFFFFFF0
set [0x0000] 0x00403202
set pc 0x0
n 1
registers
```

Ahí r3 me dio 0x00000000, no el desplazamiento esperado. Es decir, la máquina ignoró el
operando que puse en rs y tomó rt (que estaba en 0), desplazando 0. Recién cuando puse el
valor en rt (palabra 0x00023202) SRA funcionó bien. O sea, en la máquina SRA usa R[rt], igual
que SLL y SRL, y no R[rs] como figura en el manual.

## Conclusiones
Los tres desplazamientos funcionan. Con el mismo dato (0xFFFFFFF0), SRL rellena con ceros
(0x0FFFFFFF) y SRA con el bit de signo (0xFFFFFFFF).

La única diferencia con el manual está en SRA: la doc dice que opera sobre R[rs], pero la
máquina lo hace sobre R[rt].

---

# Caso 6: ANDI, ORI y XORI (lógicas con inmediato)

## Descripción
Junto las tres lógicas con inmediato en un solo caso, igual que hice con las lógicas entre
registros del Caso 4. Las tres son tipo L y comparten forma (R[rt] = R[rs] op imm). Cargo un
valor con bits conocidos y comparo cada resultado con la cuenta a mano. De paso miro la forma
que opera sobre la parte superior de la palabra, propia del formato L.

## Instructions
- ANDI r1, r2, imm → r1 = r2 & imm
- ORI  r1, r2, imm → r1 = r2 | imm
- XORI r1, r2, imm → r1 = r2 ^ imm

Tipo L: una variante del tipo I pensada para las lógicas. Como es un desperdicio usar un
inmediato de 17 bits para un AND o un OR, se achica a 16 bits y con lo que sobra la instrucción
puede operar sobre la parte inferior o sobre la parte superior de la palabra:

```
[31:27] opcode | [26:22] rs | [21:17] rt | [15:0] imm (16 bits)
```

El inmediato va con extensión por ceros, no con signo. Uso rs=r2, rt=r1 e imm=0xA, en la forma
que opera sobre la parte inferior, así que la palabra es igual salvo el opcode:
- ANDI → 00100 → 0x2082000A
- ORI  → 00101 → 0x2882000A
- XORI → 00110 → 0x3082000A

## Precondiciones
- Cargo el operando una sola vez: `set r2 0xC`. Lo elijo por sus bits, igual que en el Caso 4:
  0xC = 1100 y 0xA = 1010, así se ve claro qué hace cada operación.
- Voy cambiando la instrucción en 0x0000 y rebobino con `set pc 0x0` antes de cada `n 1`.

## Code
```
set r2 0xC

set [0x0000] 0x2082000A   # ANDI
set pc 0x0
n 1
registers

set [0x0000] 0x2882000A   # ORI
set pc 0x0
n 1
registers

set [0x0000] 0x3082000A   # XORI
set pc 0x0
n 1
registers
```

## Postcondiciones
Miro r1 con `registers` después de cada una (0xC = 1100, 0xA = 1010):
- ANDI: 1100 & 1010 = 1000 → r1 = 0x00000008
- ORI:  1100 | 1010 = 1110 → r1 = 0x0000000E
- XORI: 1100 ^ 1010 = 0110 → r1 = 0x00000006

En las tres r2 quedó intacto y CAUSE en 0.

## Parte inferior y superior
Aparte probé la forma de ANDI que opera sobre la parte superior. Ahí el inmediato se coloca en la
mitad superior de la palabra: ANDI/H r1, r2, 0x1234 (palabra 0x20831234) con r2=0xFFFFFFFF me dio
r1 = 0x12340000, es decir 0xFFFFFFFF & 0x12340000. La misma cuenta con la forma que opera sobre la
parte inferior (palabra 0x20821234) sobre r2=0xFFFFFFFF dio r1 = 0x00001234: el inmediato entra
por la parte inferior con extensión por ceros y la parte superior se descarta.

## Conclusiones
Anduvieron las tres y dieron lo que da la cuenta bit a bit. Las dos formas del formato L
funcionan: una opera sobre la parte inferior (con extensión por ceros, descartando la superior)
y la otra sobre la parte superior. Ninguna tocó los operandos ni tiró excepción.

---

# Caso 7: SLTI y SLTIU (comparación con inmediato)

## Descripción
SLTI y SLTIU comparan un registro con una constante inmediata y dejan 1 o 0 en el destino según
si es menor: R[rt] = 1 si R[rs] < imm, y 0 si no. La única diferencia entre las dos es cómo
interpretan los números: SLTI los toma con signo y SLTIU sin signo. Las pruebo juntas usando un
operando negativo, que es justo donde se nota la diferencia.

## Instructions
- SLTI  r1, r2, imm → r1 = (r2 < imm) ? 1 : 0   (con signo)
- SLTIU r1, r2, imm → r1 = (r2 < imm) ? 1 : 0   (sin signo)

Son tipo I: opcode, dos registros y un inmediato de 17 bits con signo (no tienen func ni un
tercer registro como las tipo R):

```
[31:27] opcode | [26:22] rs | [21:17] rt | [16:0] imm (17 bits, complemento a dos)
```

Uso rs=r2, rt=r1 e imm=10 (0xA). Cambia solo el opcode:
- SLTI  → 10110 → 0xB082000A
- SLTIU → 10111 → 0xB882000A

## Precondiciones
- Cargo el valor a comparar en r2 y voy cambiando la instrucción en 0x0000.
- Rebobino con `set pc 0x0` antes de cada `n 1`.
- Para el caso que da falso cargo r1 con un valor distinto de 0 (0xAA), así verifico que la
  instrucción lo pise con 0 en vez de dejarlo como estaba.

## Code
```
# SLTI (con signo)
set r2 0x5
set [0x0000] 0xB082000A
set pc 0x0
n 1
registers                 # 5 < 10

set r2 0xFFFFFFFF
set pc 0x0
n 1
registers                 # -1 < 10 con signo

# SLTIU (sin signo)
set r2 0x5
set [0x0000] 0xB882000A
set pc 0x0
n 1
registers                 # 5 < 10

set r1 0xAA
set r2 0xFFFFFFFF
set pc 0x0
n 1
registers                 # 0xFFFFFFFF sin signo NO es < 10
```

## Postcondiciones
Miro r1 con `registers` después de cada una:
- SLTI, r2=0x5        → r1 = 0x00000001 (5 < 10)
- SLTI, r2=0xFFFFFFFF → r1 = 0x00000001 (con signo 0xFFFFFFFF es -1, y -1 < 10)
- SLTIU, r2=0x5        → r1 = 0x00000001 (5 < 10)
- SLTIU, r2=0xFFFFFFFF → r1 = 0x00000000 (sin signo es 4294967295, no es < 10)

En el último caso r1 venía con 0xAA y la instrucción lo pisó con 0, así que SLTIU escribe el
resultado también cuando la comparación da falso. En las cuatro r2 quedó intacto y CAUSE en 0.

## Conclusiones
Las dos comparan bien y se ve clara la diferencia de signo: con r2=0xFFFFFFFF e imm=10, SLTI da
1 (lo lee como -1) y SLTIU da 0 (lo lee como un número enorme). El resultado se escribe siempre,
valga 1 o 0. Ninguna tocó r2 ni tiró excepción.

---

# Caso 8: LUI

## Descripción
LUI (Load Upper Immediate) carga una constante de 16 bits en la parte superior de un registro y
deja la parte inferior en cero. Existe porque en una instrucción de 32 bits no entra una
constante de 32 bits, así que para armar un número grande se hace en dos pasos: LUI mete la
mitad de arriba y después un ORI rellena la de abajo. En este caso pruebo solo la primera mitad:
que LUI coloque bien el inmediato arriba y ponga ceros abajo.

## Instructions
- LUI r1, imm → r1 = imm << 16   (el inmediato queda en los 16 bits superiores, los inferiores en 0)

Es tipo L, pero solo usa el registro destino (rt) y el inmediato; rs no interviene:

```
[31:27] opcode=00111 | [26:22] rs (sin uso) | [21:17] rt | [16] h | [15:0] imm
```

Uso rt=r1. Las palabras que armé:
- LUI r1, 0x1234 → 0x38021234
- LUI r1, 0xFFFF → 0x3802FFFF

## Precondiciones
- Cargo r1 con un valor distinto de 0 (0xAA) para confirmar que LUI lo pisa por completo,
  incluida la parte inferior, que tiene que quedar en cero.
- Cargo la instrucción en 0x0000 y rebobino con `set pc 0x0` antes de cada `n 1`.

## Code
```
set r1 0xAA
set [0x0000] 0x38021234
set pc 0x0
examine 0x0000
n 1
registers          # r1 = 0x12340000

set [0x0000] 0x3802FFFF
set pc 0x0
n 1
registers          # r1 = 0xFFFF0000
```

## Postcondiciones
Miro r1 con `registers`:
- LUI r1, 0x1234 → r1 = 0x12340000
- LUI r1, 0xFFFF → r1 = 0xFFFF0000

En los dos casos la parte inferior quedó en cero y CAUSE en 0.

## Conclusiones
Anduvo. LUI coloca el inmediato en la parte superior del registro y deja la inferior en cero, y
pisó el 0xAA que había cargado antes. Es la primera mitad de la receta para armar una constante
de 32 bits (después iría un ORI para la parte inferior). El bit h del formato L acá no cambia
nada (probé la misma palabra con h=1 y dio igual), lo que tiene sentido porque LUI siempre
carga arriba. No tiró ninguna excepción.

---

# Caso 9: SLT y SLTU (comparación entre registros)

## Descripción
SLT y SLTU son la versión tipo R de las SLTI/SLTIU del Caso 7: en vez de comparar un registro
contra un inmediato, comparan dos registros. Dejan 1 o 0 en el destino según si el primero es
menor: R[rd] = 1 si R[rs] < R[rt], y 0 si no. Igual que antes, SLT compara con signo y SLTU sin
signo. Las pruebo juntas con un operando negativo, que es donde se ve la diferencia.

## Instructions
- SLT  r3, r1, r2 → r3 = (r1 < r2) ? 1 : 0   (con signo)
- SLTU r3, r1, r2 → r3 = (r1 < r2) ? 1 : 0   (sin signo)

Son tipo R con los mismos campos que ADD (rd=r3, rs=r1, rt=r2); cambia solo el func:
- SLT  → func 001100 → 0x0044300C
- SLTU → func 001101 → 0x0044300D

## Precondiciones
- Cargo el valor a comparar en r1 y el límite en r2 (0xA = 10).
- Cambio la instrucción en 0x0000 y vuelvo el PC a cero con `set pc 0x0` antes de cada `n 1`.
- Para el caso que da falso precargo r3 con 0xAA, así verifico que la instrucción lo pise con 0.

## Code
```
# SLT (con signo)
set r1 0x5
set r2 0xA
set [0x0000] 0x0044300C
set pc 0x0
n 1
registers                 # 5 < 10

set r1 0xFFFFFFFF
set pc 0x0
n 1
registers                 # -1 < 10 con signo

# SLTU (sin signo)
set r1 0x5
set [0x0000] 0x0044300D
set pc 0x0
n 1
registers                 # 5 < 10

set r3 0xAA
set r1 0xFFFFFFFF
set pc 0x0
n 1
registers                 # 0xFFFFFFFF sin signo NO es < 10
```

## Postcondiciones
Miro r3 con `registers` después de cada una:
- SLT, r1=0x5        → r3 = 0x00000001 (5 < 10)
- SLT, r1=0xFFFFFFFF → r3 = 0x00000001 (con signo 0xFFFFFFFF es -1, y -1 < 10)
- SLTU, r1=0x5        → r3 = 0x00000001 (5 < 10)
- SLTU, r1=0xFFFFFFFF → r3 = 0x00000000 (sin signo es 4294967295, no es < 10)

En el último r3 traía 0xAA y quedó en 0, o sea la escritura se hace aunque el resultado sea
falso. r1 y r2 no se tocaron en ninguna y CAUSE quedó en 0.

## Conclusiones
Se portan igual que las SLTI/SLTIU del Caso 7, solo que comparando dos registros: los cuatro
resultados coincidieron con la cuenta y volvió a aparecer el tema del signo en el caso de
0xFFFFFFFF (SLT lo trata como -1 y da 1, SLTU como un número grande y da 0). No tocaron r1 ni r2
ni tiraron excepción.

---

# Caso 10: LW y SW (load y store de palabra)

## Descripción
LW y SW son la primera pareja que toca memoria: SW (store word) escribe una palabra de un
registro a memoria y LW (load word) la trae de vuelta a un registro. Las dos calculan la
dirección de la misma forma, sumando un registro base y un offset. La idea del caso es guardar un
valor con SW, leerlo con LW y ver que vuelve intacto, y de paso comprobar que el offset se suma a
la base.

## Instructions
- SW rt, offset(rs) → MEM[R[rs] + offset] = R[rt]
- LW rt, offset(rs) → R[rt] = MEM[R[rs] + offset]

Son tipo I. rs es el registro base (la dirección), rt es el dato que se guarda o se carga, y el
inmediato de 17 bits es el offset:

```
[31:27] opcode | [26:22] rs (base) | [21:17] rt (dato) | [16:0] offset (17 bits)
```

Uso r2 como base, apuntándola a 0x100: es una dirección alineada a 4 bytes y bien lejos del
código, que está en 0x0000. Las palabras:
- SW r1, 0(r2) → 0x48820000   (opcode SW 01001)
- LW r3, 0(r2) → 0x40860000   (opcode LW 01000)
- SW r1, 4(r2) → 0x48820004
- LW r3, 4(r2) → 0x40860004

## Precondiciones
- Apunto la base: `set r2 0x100`.
- Cargo el dato a guardar en r1 y cambio la instrucción en 0x0000, rebobinando con `set pc 0x0`
  antes de cada `n 1`.
- Antes de cada LW pongo r3 en 0, así veo que la carga realmente lo pisa con el valor leído.

## Code
```
# SW: guardar r1 en MEM[r2+0]
set r2 0x100
set r1 0x12345678
set [0x0000] 0x48820000     # SW r1, 0(r2)
set pc 0x0
n 1
examine 0x100

# LW: leer MEM[r2+0] en r3
set r3 0x0
set [0x0000] 0x40860000     # LW r3, 0(r2)
set pc 0x0
n 1
registers

# offset distinto de 0: guardar en MEM[r2+4] y leerlo
set r1 0x11112222
set [0x0000] 0x48820004     # SW r1, 4(r2)
set pc 0x0
n 1
examine 0x104

set r3 0x0
set [0x0000] 0x40860004     # LW r3, 4(r2)
set pc 0x0
n 1
registers
```

## Postcondiciones
- SW r1, 0(r2) con r1=0x12345678 → `examine 0x100` devuelve 0x12345678
- LW r3, 0(r2) → r3 = 0x12345678 (el mismo valor que había guardado)
- SW r1, 4(r2) con r1=0x11112222 → `examine 0x104` devuelve 0x11112222
- LW r3, 4(r2) → r3 = 0x11112222

En ningún acceso apareció un CAUSE distinto de 0, así que no hubo fallo de alineación.

## Conclusiones
Anduvieron las dos. Lo que guardo con SW lo recupero igual con LW, o sea el par escribe y lee
memoria bien. El offset se suma a la base como corresponde: con base 0x100, el offset 0 fue a
parar a 0x100 y el offset 4 a 0x104, cada valor en su dirección.

---

# Caso 11: SH, LH y LHU (media palabra)

## Descripción
La versión de 16 bits (media palabra) del par LW/SW. SH (store half) guarda los 16 bits bajos de
un registro en memoria; LH (load half) trae 16 bits extendiéndolos con signo y LHU (load half
unsigned) los trae extendiéndolos con ceros. Uso un valor con el bit 15 en 1 para ver la
diferencia entre LH y LHU, que es donde entra el signo.

## Instructions
- SH  rt, offset(rs) → MEM[R[rs]+offset] (16 bits) = R[rt] (los 16 bits bajos)
- LH  rt, offset(rs) → R[rt] = extensión con signo de los 16 bits leídos
- LHU rt, offset(rs) → R[rt] = extensión con ceros de los 16 bits leídos

Tipo I, mismo formato que LW/SW: rs base, rt el dato, offset de 17 bits. Uso r2=0x100 de base.
Opcodes SH 01010, LH 01100, LHU 01101:
- SH  r1, 0(r2) → 0x50820000
- LH  r3, 0(r2) → 0x60860000
- LHU r3, 0(r2) → 0x68860000

## Precondiciones
- Base en r2=0x100. Antes de guardar limpio la palabra con un SW de 0, así el `examine` se lee
  sin restos de pruebas anteriores.
- Guardo 0xABCD (bit 15 en 1) con SH y después lo leo con LH y con LHU.
- Antes de cada carga pongo r3 en 0 para ver que la instrucción lo pise.

## Code
```
# limpio MEM[0x100]
set r2 0x100
set r1 0x0
set [0x0000] 0x48820000     # SW r1, 0(r2)
set pc 0x0
n 1

# SH: guardar los 16 bits bajos de r1
set r1 0xABCD
set [0x0000] 0x50820000     # SH r1, 0(r2)
set pc 0x0
n 1
examine 0x100

# LH: cargar con signo
set r3 0x0
set [0x0000] 0x60860000     # LH r3, 0(r2)
set pc 0x0
n 1
registers

# LHU: cargar sin signo
set r3 0x0
set [0x0000] 0x68860000     # LHU r3, 0(r2)
set pc 0x0
n 1
registers
```

## Postcondiciones
- SH r1(0xABCD), 0(r2) → `examine 0x100` = 0x0000ABCD (guardó los 16 bits, la parte alta quedó en 0)
- LH r3, 0(r2)  → r3 = 0xFFFFABCD, y la operación de memoria fue de 2 bytes (Size = 2)
- LHU r3, 0(r2) → r3 = 0x0000ABCD, también con Size = 2

CAUSE quedó en 0 en todos los accesos.

## Conclusiones
Andan las tres. SH guarda los 16 bits bajos en memoria, y las dos cargas leyeron 2 bytes como
corresponde; la diferencia entre ellas se ve justo donde tiene que verse: 0xABCD tiene el bit
15 en 1, así que LH lo extendió con signo (0xFFFFABCD) y LHU lo rellenó con ceros (0x0000ABCD).

---

# Caso 12: SB, LB y LBU (byte)

## Descripción
Lo mismo que el Caso 11 pero con bytes: SB guarda el byte bajo de un registro, y LB y LBU lo
leen de vuelta, uno rellenando con el signo y el otro con ceros. Guardo 0xCD (bit 7 en 1) para
que las dos cargas den distinto, que es donde se ve si cada una extiende como corresponde.

## Instructions
- SB  rt, offset(rs) → MEM[R[rs]+offset] (8 bits) = R[rt] (el byte bajo)
- LB  rt, offset(rs) → R[rt] = extensión con signo del byte leído
- LBU rt, offset(rs) → R[rt] = extensión con ceros del byte leído

Tipo I, el mismo formato que las de palabra y media palabra. Uso r2=0x100 de base. Opcodes SB
01011, LB 01110, LBU 01111:
- SB  r1, 0(r2) → 0x58820000
- LB  r3, 0(r2) → 0x70860000
- LBU r3, 0(r2) → 0x78860000

## Precondiciones
- Base en r2=0x100 y limpio la palabra con un SW de 0 antes de guardar.
- Guardo 0xCD (bit 7 en 1) con SB y lo leo con LB y con LBU.
- Antes de cada carga pongo r3 en 0.

## Code
```
# limpio MEM[0x100]
set r2 0x100
set r1 0x0
set [0x0000] 0x48820000     # SW r1, 0(r2)
set pc 0x0
n 1

# SB: guardar el byte bajo de r1
set r1 0xCD
set [0x0000] 0x58820000     # SB r1, 0(r2)
set pc 0x0
n 1
examine 0x100

# LB: cargar con signo
set r3 0x0
set [0x0000] 0x70860000     # LB r3, 0(r2)
set pc 0x0
n 1
registers

# LBU: cargar sin signo
set r3 0x0
set [0x0000] 0x78860000     # LBU r3, 0(r2)
set pc 0x0
n 1
registers
```

## Postcondiciones
- SB r1(0xCD), 0(r2) → `examine 0x100` = 0x000000CD (guardó un byte, el resto quedó en 0)
- LB r3, 0(r2)  → r3 = 0xFFFFFFCD (extendió el signo, el byte 0xCD tiene el bit 7 en 1)
- LBU r3, 0(r2) → r3 = 0x000000CD (extendió con ceros)

Las dos cargas leyeron 1 byte (Size = 1) y CAUSE quedó en 0.

## Conclusiones
Andan las tres. SB guarda el byte bajo, LB lo lee extendiendo con signo (0xCD → 0xFFFFFFCD) y LBU
extendiendo con ceros (0xCD → 0x000000CD). Es el mismo comportamiento que las versiones de media
palabra del Caso 11, pero a nivel byte: el ancho del acceso fue de 1 byte (Size = 1) y el signo
lo decide el bit 7 del byte leído.

---

# Caso 13: LWX (load word indexed)

## Descripción
LWX es un load de palabra pero con la dirección indexada: en vez de base + un offset inmediato,
la dirección sale de sumar dos registros. Según el manual (Tabla A.2) hace
R[rt] = MEM[R[rs] + R[rd]], o sea suma base más índice y trae la palabra al destino, que acá es
el campo rt (el índice va en rd). La idea del caso es dejar un valor conocido en memoria,
apuntarlo con base + índice y ver que la carga lo traiga al registro destino.

## Instructions
- LWX rt, rs, rd → R[rt] = MEM[R[rs] + R[rd]]

Es tipo R con func 010100. Uso rs=r4 (base), rd=r5 (índice) y rt=r3 (destino):

```
opcode 00000 | rs=r4 (00100) | rt=r3 (00011) | rd=r5 (00101) | aux 00000 | X 0 | func 010100
```

Pegando todo y pasándolo a hexa me queda 0x01065014.

## Precondiciones
- Primero dejo un valor conocido en memoria: guardo 0x0000ABCD en la dirección 0x100 con un SW
  (r2=0x100 de base, r1=0xABCD el dato). Verifico con `examine 0x100` que quedó cargado.
- Para la carga armo la dirección como base + índice = 0x100: `set r4 0xC0` (base) y
  `set r5 0x40` (índice), que suman 0x100.
- Pongo r3 en 0 para ver que LWX lo pise con el valor leído.
- Cargo la instrucción en 0x0000 y rebobino con `set pc 0x0` antes de `n 1`.

## Code
```
# dejar 0x0000ABCD en MEM[0x100]
set r1 0x0000ABCD
set r2 0x100
set [0x0000] 0x48820000     # SW r1, 0(r2)
set pc 0x0
n 1
examine 0x100               # 0x0000ABCD

# LWX r3, r4, r5  → R[3] = MEM[r4 + r5] = MEM[0xC0 + 0x40] = MEM[0x100]
set r4 0xC0
set r5 0x40
set r3 0x0
set [0x0000] 0x01065014
set pc 0x0
n 1
registers
```

## Postcondiciones
Como r4+r5 = 0xC0 + 0x40 = 0x100 y ahí guardé 0x0000ABCD, después de `n 1` el registro destino
r3 quedó en 0x0000ABCD. La operación de memoria reporta la dirección y el ancho esperados
(Address = 0x100, Size = 4), CAUSE quedó en 0 y r4/r5 no cambiaron.

## Conclusiones
Anda. LWX armó la dirección sumando base más índice (0xC0 + 0x40 = 0x100), leyó la palabra
entera de ahí y la dejó en el registro destino. Lo único para tener presente al codificarla es
la repartija de campos: el destino va en rt y el índice en rd, como marca la fórmula de la
Tabla A.2.

---

# Caso 14: LHX, LHUX, LBX y LBUX (cargas indexadas de media palabra y byte)

## Descripción
Son las cargas indexadas hermanas de LWX (Caso 13), pero de media palabra y de byte: LHX y LHUX
traen 16 bits (con signo y sin signo) y LBX y LBUX traen 8 bits (con signo y sin signo), siempre
con la dirección armada como base + índice, R[rt] = MEM[R[rs] + R[rd]]. Las junto en un caso
porque comparten forma, y lo que más me interesa es que respeten lo del signo/ceros igual que sus
versiones no indexadas (Caso 11 y 12).

## Instructions
- LHX  rt, rs, rd → R[rt] = MEM[R[rs] + R[rd]] (media palabra, extensión con signo)
- LHUX rt, rs, rd → R[rt] = MEM[R[rs] + R[rd]] (media palabra, extensión con ceros)
- LBX  rt, rs, rd → R[rt] = MEM[R[rs] + R[rd]] (byte, extensión con signo)
- LBUX rt, rs, rd → R[rt] = MEM[R[rs] + R[rd]] (byte, extensión con ceros)

Las cuatro son tipo R con los mismos campos que usé en LWX (rs=r4 base, rd=r5 índice, rt=r3
destino); cambia solo el func. Partiendo de la palabra de LWX (0x01065014, func 010100) y bajando
el func me quedaron:
- LHX  → func 010000 → 0x01065010
- LHUX → func 010001 → 0x01065011
- LBX  → func 010010 → 0x01065012
- LBUX → func 010011 → 0x01065013

## Precondiciones
- Dejo 0x0000ABCD en 0x100 con un SW (r1=0xABCD, base r2=0x100) y lo confirmo con `examine 0x100`.
  Ese valor me sirve para las cuatro: la media palabra baja es 0xABCD (bit 15 en 1) y el byte bajo
  es 0xCD (bit 7 en 1), así se ve la diferencia entre las versiones con signo y sin signo.
- Para la carga apunto la dirección como base + índice = 0xC0 + 0x40 = 0x100: `set r4 0xC0` y
  `set r5 0x40`.
- Pongo r3 en 0 para ver que la carga lo pise, y vuelvo a setear r4, r5 y r3 antes de cada una.
  Rebobino con `set pc 0x0` antes de cada `n 1`.

## Code
```
# dejar 0x0000ABCD en MEM[0x100]
set r1 0x0000ABCD
set r2 0x100
set [0x0000] 0x48820000
set pc 0x0
n 1
examine 0x100               # 0x0000ABCD

# LHX r3, r4, r5  (y lo mismo cambiando la palabra por LHUX/LBX/LBUX)
set r4 0xC0
set r5 0x40
set r3 0x0
set [0x0000] 0x01065010     # LHX  (01065011 LHUX, 01065012 LBX, 01065013 LBUX)
set pc 0x0
n 1
registers
```

## Postcondiciones
Con base 0xC0 e índice 0x40 la dirección da 0x100, y r3 quedó con el valor de ahí en las cuatro:
- LHX  → r3 = 0xFFFFABCD (Size = 2)
- LHUX → r3 = 0x0000ABCD (Size = 2)
- LBX  → r3 = 0xFFFFFFCD (Size = 1)
- LBUX → r3 = 0x000000CD (Size = 1)

En las cuatro la operación de memoria fue sobre Address = 0x100, CAUSE quedó en 0 y r4/r5 no
cambiaron.

## Conclusiones
Andan las cuatro. Todas armaron bien la dirección base + índice y leyeron el ancho que les
corresponde (2 bytes las de media palabra, 1 byte las de byte). Y el signo se comporta igual
que en las versiones no indexadas: LHX y LBX extienden con el bit alto del dato (que acá era 1)
y LHUX y LBUX rellenan con ceros.

---

# Caso 15: MUL, MULH y MULHU (multiplicación)

## Descripción
Las tres formas de multiplicar dos registros. El producto de dos números de 32 bits puede necesitar
hasta 64 bits, así que la operación se parte en dos mitades: MUL devuelve la parte baja (los 32 bits
de abajo del producto) y MULH/MULHU la parte alta. La diferencia entre MULH y MULHU es el signo:
MULH toma los operandos con signo y MULHU sin signo. Las pruebo juntas con tres pares de valores:
uno chico que entra en 32 bits, uno que desborda para que la parte alta no sea cero, y uno con
negativos para que se note la diferencia entre MULH y MULHU.

## Instructions
- MUL   r3, r1, r2 → r3 = parte baja de r1 * r2
- MULH  r3, r1, r2 → r3 = parte alta de r1 * r2 (con signo)
- MULHU r3, r1, r2 → r3 = parte alta de r1 * r2 (sin signo)

Las tres son tipo R con los mismos campos que ADD (rs=r1, rt=r2, rd=r3); cambia solo el func:
- MUL   → func 010101 → 0x00443015
- MULH  → func 010110 → 0x00443016
- MULHU → func 010111 → 0x00443017

## Precondiciones
- Cargo los dos factores en r1 y r2 y voy cambiando la instrucción en 0x0000, rebobinando con
  `set pc 0x0` antes de cada `n 1`.
- Uso tres pares: 7 y 3 (producto chico), 0x10000 y 0x10000 (0x10000 * 0x10000 = 0x100000000, que
  ya no entra en 32 bits) y 0xFFFFFFFF con 0xFFFFFFFF (o sea -1 por -1 si se miran con signo).

## Code
```
# 7 * 3
set r1 0x7
set r2 0x3
set [0x0000] 0x00443015     # MUL   (00443016 MULH, 00443017 MULHU)
set pc 0x0
n 1
registers

# 0x10000 * 0x10000 (desborda 32 bits) y 0xFFFFFFFF * 0xFFFFFFFF (-1 * -1), igual con las tres palabras
```

## Postcondiciones
Miro r3 con `registers` después de cada una:
- 7 * 3          → MUL = 0x00000015 (21), MULH = 0x00000000, MULHU = 0x00000000
- 0x10000 * 0x10000 → MUL = 0x00000000, MULH = 0x00000001, MULHU = 0x00000001
- 0xFFFFFFFF * 0xFFFFFFFF → MUL = 0x00000001, MULH = 0x00000000, MULHU = 0xFFFFFFFE

En las tres r1 y r2 quedaron intactos y CAUSE en 0.

## Conclusiones
Andan las tres. Con el producto chico (7*3=21) la parte baja da bien y la alta queda en cero. Con
0x10000*0x10000 el resultado no entra en 32 bits: MUL da 0 (la parte baja) y la parte alta aparece
en MULH/MULHU con un 1, así que entre las dos mitades se arma el 0x100000000. Y con 0xFFFFFFFF por
0xFFFFFFFF se ve la diferencia de signo: mirándolo con signo es -1 por -1 = 1, y ahí MUL da 1 y MULH
da 0; mirándolo sin signo el producto es enorme y su parte alta es 0xFFFFFFFE, que es lo que da
MULHU. Ninguna tocó los operandos ni tiró excepción.

---

# Caso 16: DIV, DIVU, REST y RESTU (división)

## Descripción
La división entera partida en cociente y resto: DIV da el cociente y REST el resto de dividir dos
registros, y DIVU/RESTU son las mismas pero tomando los números sin signo. Las junto en un caso.
Uso una división que no es exacta (20 entre 3) para ver cociente y resto a la vez, la repito con el
dividendo negativo para la diferencia con/sin signo, y de paso pruebo qué pasa al dividir por cero.

## Instructions
- DIV   r3, r1, r2 → r3 = r1 / r2 (cociente, con signo)
- DIVU  r3, r1, r2 → r3 = r1 / r2 (cociente, sin signo)
- REST  r3, r1, r2 → r3 = resto de r1 / r2 (con signo)
- RESTU r3, r1, r2 → r3 = resto de r1 / r2 (sin signo)

Tipo R, mismos campos que ADD (rs=r1, rt=r2, rd=r3); cambia solo el func:
- DIV   → func 011000 → 0x00443018
- DIVU  → func 011001 → 0x00443019
- REST  → func 011010 → 0x0044301A
- RESTU → func 011011 → 0x0044301B

## Precondiciones
- Cargo dividendo en r1 y divisor en r2, cambio la instrucción en 0x0000 y rebobino con
  `set pc 0x0` antes de cada `n 1`.
- Primer par: 20 (0x14) y 3. Segundo par: -20 (0xFFFFFFEC) y 3, para ver la diferencia de signo.
- Para la división por cero pongo r2 en 0 y precargo r3 con 0xAA, así veo si la instrucción lo pisa
  o lo deja como estaba.

## Code
```
# 20 / 3
set r1 0x14
set r2 0x3
set [0x0000] 0x00443018     # DIV   (00443019 DIVU, 0044301A REST, 0044301B RESTU)
set pc 0x0
n 1
registers

# -20 / 3 (dividendo 0xFFFFFFEC), igual con las cuatro palabras

# division por cero
set r1 0x7
set r2 0x0
set r3 0xAA
set [0x0000] 0x00443018     # DIV
set pc 0x0
n 1
registers
```

## Postcondiciones
Miro r3 con `registers`:
- 20 / 3   → DIV = 0x00000006 (6), DIVU = 0x00000006, REST = 0x00000002 (2), RESTU = 0x00000002
- -20 / 3  → DIV = 0xFFFFFFFA (-6), REST = 0xFFFFFFFE (-2); sin signo DIVU = 0x5555554E y RESTU = 0x00000002
- 7 / 0    → r3 quedó en 0xAA (no lo pisó) y CAUSE pasó a 0x00000003; lo mismo con REST

En las divisiones normales r1 y r2 quedaron intactos y CAUSE en 0.

## Conclusiones
Andan las cuatro. Con 20/3 el cociente da 6 y el resto 2, tanto con signo como sin signo (los dos
números son positivos, así que dan igual). Con el dividendo negativo se ve la diferencia: DIV lee
-20 y da cociente -6 con resto -2, mientras que DIVU/RESTU leen 0xFFFFFFEC como un número grande y
dan otra cosa (0x5555554E y 2). El cociente y el resto son coherentes entre sí en los dos casos.

Aparte probé dividir por cero: ahí la máquina no completa la operación, deja el destino como estaba
(r3 se quedó en 0xAA) y levanta una excepción (CAUSE = 0x00000003). O sea no devuelve un resultado
inventado, corta con una excepción.

---

# Caso 17: BEQ, BNE, BLT, BGT, BLE y BGE (saltos condicionales)

## Descripción
Los seis saltos condicionales. Cada uno compara dos registros y, si se cumple la condición, salta a
una dirección relativa; si no, sigue con la instrucción de al lado. Se diferencian solo en la
condición: BEQ salta si son iguales, BNE si son distintos, y BLT/BGT/BLE/BGE según el orden (menor,
mayor, menor o igual, mayor o igual). Los junto en un caso porque comparten forma y lo que se mira
en todos es lo mismo: a dónde queda el PC después de ejecutar. Pruebo cada uno con un caso que salta
y otro que no, y de paso miro si la comparación es con signo y si el salto también va para atrás.

## Instructions
- BEQ rs, rt, offset → salta si R[rs] == R[rt]
- BNE rs, rt, offset → salta si R[rs] != R[rt]
- BLT rs, rt, offset → salta si R[rs] <  R[rt]
- BGT rs, rt, offset → salta si R[rs] >  R[rt]
- BLE rs, rt, offset → salta si R[rs] <= R[rt]
- BGE rs, rt, offset → salta si R[rs] >= R[rt]

Son tipo I: opcode, los dos registros a comparar y un offset de 17 bits con signo.

```
[31:27] opcode | [26:22] rs | [21:17] rt | [16:0] offset (17 bits, complemento a dos)
```

El offset se cuenta en palabras y desde la instrucción siguiente al salto. Uso rs=r1, rt=r2 y un
offset de 4: si el salto se toma, el PC salta a 0x0014 (la siguiente es 0x0004, más 4 palabras son
16 bytes); si no se toma, el PC sigue en 0x0004. Cambia solo el opcode:
- BEQ → 10000 → 0x80440004
- BNE → 10001 → 0x88440004
- BLT → 10010 → 0x90440004
- BGT → 10011 → 0x98440004
- BLE → 10100 → 0xA0440004
- BGE → 10101 → 0xA8440004

## Precondiciones
- Cargo los dos valores a comparar en r1 y r2, cambio la instrucción en 0x0000 y rebobino con
  `set pc 0x0` antes de cada `n 1`.
- Para cada salto uso un par que lo hace saltar y otro que no. Con 5 y 8 alcanza para casi todos;
  para el igual/desigual uso 5 y 5.
- Para ver si compara con signo pruebo BLT y BGE con r1 = 0xFFFFFFFF (que es -1) y r2 = 1.

## Code
```
# BEQ con r1 == r2 (salta) y con r1 != r2 (no salta). Igual con las otras cambiando la palabra.
set r1 0x5
set r2 0x5
set [0x0000] 0x80440004     # BEQ
set pc 0x0
n 1
registers                   # PC = 0x14 (saltó)

set r1 0x5
set r2 0x8
set pc 0x0
n 1
registers                   # PC = 0x04 (no saltó)
```

## Postcondiciones
Miro el PC con `registers`. En cada uno, cuando se cumple la condición el PC queda en 0x00000014 y
cuando no, en 0x00000004:
- BEQ: 5,5 saltó (0x14); 5,8 no (0x04)
- BNE: 5,8 saltó; 5,5 no
- BLT: 5,8 saltó (5 < 8); 8,5 no
- BGT: 8,5 saltó (8 > 5); 5,8 no
- BLE: 5,5 y 5,8 saltaron; 8,5 no
- BGE: 5,5 y 8,5 saltaron; 5,8 no

Con el negativo: BLT con r1=0xFFFFFFFF y r2=1 saltó, y BGE con esos mismos valores no saltó, o sea
tomó 0xFFFFFFFF como -1 (-1 < 1 y -1 no es >= 1).

Aparte probé un offset negativo: puse un BEQ que se toma en la dirección 0x20 con offset -2, y el PC
quedó en 0x1C, o sea saltó para atrás como corresponde.

## Conclusiones
Andan los seis. Cada uno salta justo cuando se cumple su condición y sigue de largo cuando no, y el
destino del salto cae donde tiene que caer (la instrucción siguiente más el offset en palabras). La
comparación es con signo, así que un 0xFFFFFFFF lo trata como -1, y el offset funciona para los dos
lados, hacia adelante y hacia atrás. Ninguno tocó los registros que comparó.

---

# Caso 18: J y JAL (saltos incondicionales)

## Descripción
Los dos saltos incondicionales de tipo J. J salta y listo; JAL además guarda la dirección de
retorno en $ra, que es lo que se usa para llamar funciones. Pruebo que cada uno vaya a donde
tiene que ir, que JAL deje bien el retorno, y que la dirección sea absoluta (saltando desde un
PC distinto de cero).

## Instructions
- J   addr → PC = addr (en palabras)
- JAL addr → r31 = PC + 4; PC = addr (en palabras)

Son tipo J: opcode y una dirección de 27 bits que se cuenta en palabras.

```
[31:27] opcode | [26:0] address (en palabras)
```

Para saltar al byte 0x40 la dirección en palabras es 0x10. Cambia solo el opcode:
- J   → 00010 → 0x10000010
- JAL → 00011 → 0x18000010

## Precondiciones
- Cargo la instrucción en 0x0000 y rebobino con `set pc 0x0` antes de cada `n 1`.
- Para ver que la dirección es absoluta, pongo otro J en 0x0040 apuntando a la palabra 0x2
  (byte 0x8) y lo ejecuto desde ahí.

## Code
```
# J a 0x40
set [0x0000] 0x10000010
set pc 0x0
n 1
registers                   # PC = 0x40

# JAL a 0x40
set [0x0000] 0x18000010
set pc 0x0
n 1
registers                   # PC = 0x40 y r31 = 0x4

# J desde 0x40 a la palabra 0x2 (byte 0x8)
set [0x0040] 0x10000002
set pc 0x40
n 1
registers                   # PC = 0x8
```

## Postcondiciones
- J con addr 0x10 → PC quedó en 0x00000040.
- JAL con addr 0x10 → PC en 0x00000040 y r31 en 0x00000004 (la dirección de la instrucción
  siguiente al salto).
- J ejecutado desde 0x40 con addr 0x2 → PC quedó en 0x00000008.

CAUSE en 0 en los tres.

## Conclusiones
Andan los dos. La dirección va en palabras y es absoluta: saltando desde 0x40 con addr 0x2 el
PC cayó en 0x8, no en algo relativo. JAL dejó en r31 la dirección de la instrucción siguiente
(PC + 4), que es lo que después usa JR para volver.

---

# Caso 19: JR y JALR (saltos por registro)

## Descripción
Los saltos que toman el destino de un registro en vez de la instrucción. JR es el retorno
clásico de función (saltar a lo que quedó en $ra) y JALR es la versión que además guarda la
dirección de retorno, para llamadas indirectas. Pruebo que salten a donde dice el registro,
qué pasa con una dirección desalineada, y en JALR dónde queda el retorno.

## Instructions
- JR   rs → PC = R[rs]
- JALR rs → r31 = PC + 4; PC = R[rs]

Tipo R. Uso rs=r1; para JALR además probé poniendo un rd (r3). Cambia solo el func:
- JR   → func 001110 → 0x0040000E
- JALR → func 001111 → 0x0040300F (con rd=r3)

## Precondiciones
- Cargo el destino del salto en r1 (0x40), la instrucción en 0x0000 y rebobino con
  `set pc 0x0` antes de cada `n 1`.
- Para la dirección desalineada uso r1 = 0x41.

## Code
```
# JR a lo que dice r1
set r1 0x40
set [0x0000] 0x0040000E
set pc 0x0
n 1
registers                   # PC = 0x40

# JALR
set r1 0x40
set r3 0x0
set r31 0x0
set [0x0000] 0x0040300F
set pc 0x0
n 1
registers

# JR a una direccion desalineada
set r1 0x41
set [0x0000] 0x0040000E
set pc 0x0
n 2
registers
```

## Postcondiciones
- JR con r1 = 0x40 → PC quedó en 0x00000040, CAUSE en 0.
- JALR con r1 = 0x40 → PC en 0x00000040, pero el retorno no quedó donde esperaba: r31 siguió
  en 0 y el 0x00000004 apareció en r3 (el rd de la palabra).
- JR con r1 = 0x41 → el salto en sí no se quejó (PC quedó en 0x41), y al ejecutar la
  instrucción siguiente saltó la excepción de alineación: CAUSE = 0x00000005 y
  BADVADR = 0x00000041.

## Diferencia con el manual (JALR)
El manual (Tabla A.2) dice que JALR guarda el retorno en R[31], igual que JAL. Pero acá
esperaba r31 = 0x4 y r31 quedó en 0: el retorno fue a parar al registro que puse en el campo
rd (r3 = 0x00000004). O sea, en la máquina el registro de retorno de JALR se elige con rd, no
es fijo r31 como figura en el manual.

## Conclusiones
JR anda bien: salta a lo que diga el registro, y si la dirección está desalineada no falla el
salto sino la ejecución siguiente, con la excepción de alineación (CAUSE 5) y la dirección
culpable en BADVADR. JALR salta bien pero guarda el retorno en rd en vez de r31 como dice el
manual; sabiendo eso se puede usar igual, poniendo en rd el registro donde uno quiere el
retorno.

---

# Caso 20: SLLR, SRLR y SRAR (desplazamientos por registro)

## Descripción
Las versiones por registro de los desplazamientos del Caso 5: en vez de una constante fija en
la instrucción, la cantidad de bits a desplazar sale de un registro. Pruebo los tres con el
mismo valor para comparar los rellenos, y de paso qué pasa si la cuenta se pasa de 31.

## Instructions
- SLLR r3, r1, r2 → r3 = r2 << r1 (izquierda)
- SRLR r3, r1, r2 → r3 = r2 >> r1 lógico (rellena con 0)
- SRAR r3, r1, r2 → r3 = r2 >> r1 aritmético (rellena con el bit de signo)

Tipo R, mismos campos que ADD: la cuenta va en rs (r1), el valor a desplazar en rt (r2) y el
resultado en rd (r3). De la cuenta se usan solo los 5 bits bajos. Cambia solo el func:
- SLLR → func 000011 → 0x00443003
- SRLR → func 000100 → 0x00443004
- SRAR → func 000101 → 0x00443005

## Precondiciones
- Cargo la cuenta en r1 (4) y el valor en r2. Uso 0x80000010, que tiene el bit de signo
  prendido y un bit abajo, así con una sola palabra se ven las tres cosas: lo que se pierde
  por arriba en SLLR y la diferencia de relleno entre SRLR y SRAR.
- Para la cuenta pasada de 31 uso r1 = 36 (0x24).
- Rebobino con `set pc 0x0` antes de cada `n 1`.

## Code
```
# SLLR: 0x80000010 << 4
set r1 0x4
set r2 0x80000010
set [0x0000] 0x00443003     # 00443004 SRLR, 00443005 SRAR
set pc 0x0
n 1
registers

# cuenta 36 (se pasa de 31)
set r1 0x24
set [0x0000] 0x00443003
set pc 0x0
n 1
registers
```

## Postcondiciones
Leo r3 con `registers`:
- SLLR con cuenta 4 → r3 = 0x00000100 (el bit de signo se fue por arriba)
- SRLR con cuenta 4 → r3 = 0x08000001 (rellena con ceros)
- SRAR con cuenta 4 → r3 = 0xF8000001 (rellena con el bit de signo)
- SLLR con cuenta 36 → r3 = 0x00000100, lo mismo que con 4

CAUSE quedó en 0 y r1/r2 intactos en todas.

## Conclusiones
Andan los tres, con la cuenta en rs y el valor en rt como dice el manual (acá no apareció el
problema de operandos del SRA del Caso 5). Los rellenos dan igual que en las versiones con
constante: SRLR mete ceros y SRAR arrastra el signo. Y de la cuenta se usan solo los 5 bits
bajos: con 36 desplazó 4 (36 = 32 + 4), tal cual lo describe el manual con R[rs][4:0].

