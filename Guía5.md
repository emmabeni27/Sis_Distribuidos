https://chatgpt.com/share/6aa2baf0-a05c-83e9-8e23-2e7f207827ae

## Nivel 1: fundamentos y reconocimiento

1) **ATOMICIDAD**: Una transacción se ejecuta completamente o no se ejecuta nada.

Si todas las operaciones de la transacción salen bien → se confirman.
Si alguna falla → se hace rollback y se deshacen las anteriores.

Problema que resuelve: evita que una transacción quede a medias.

Ejemplo: transferir $10.000 de A a B. No puede ocurrir que se descuenten de A pero nunca se acrediten en B.

**CONSISTENCIA**: Una transacción debe llevar la base de datos de un estado válido a otro estado válido, respetando todas sus restricciones e invariantes.

Problema que resuelve: evita que una transacción deje la base de datos en un estado inválido o incoherente.

Ejemplo: si existe una regla que impide que una cuenta tenga saldo negativo, una transacción no debería terminar violando esa regla.

**AISLAMIENTO**: Las transacciones concurrentes deben comportarse, según el nivel de aislamiento, como si no interfirieran incorrectamente entre sí.

Problema que resuelve: evita que las transacciones concurrentes produzcan interferencias o resultados incorrectos.

Ejemplo: dos transacciones intentan modificar el mismo saldo al mismo tiempo. El aislamiento evita que una lea/modifique datos intermedios de la otra de una manera que produzca un resultado incorrecto.

**DURABILIDAD**: Una vez que una transacción fue confirmada (commit), sus cambios deben persistir, incluso si después ocurre una falla del sistema.

Problema que resuelve: evita la pérdida de datos ya confirmados ante crashes, cortes de energía, etc.

Ejemplo: si el banco confirma una transferencia y justo después se cae el servidor, la transferencia no debería desaparecer.

| Propiedad        | Garantiza                                             | Problema que evita             |
| ---------------- | ----------------------------------------------------- | ------------------------------ |
| **Atomicidad**   | Todo o nada                                           | Transacciones incompletas      |
| **Consistencia** | Estado válido → estado válido                         | Datos/invariantes inválidos    |
| **Aislamiento**  | Correcta interacción entre transacciones concurrentes | Interferencias y anomalías     |
| **Durabilidad**  | Los commits sobreviven a fallos                       | Pérdida de cambios confirmados |

A → ¿Se hizo todo o nada?
C → ¿La BD quedó válida?
I → ¿Las transacciones se molestaron entre sí?
D → ¿Lo confirmado sobrevivió a una falla?

2) Atomicidad se refiere a una misma transacción, mientras que aislamiento se refiere a la interacción entre varias trnsacciones concurrentes.

Ejemplo: atomicidad preservada, aislamiento no

Supongamos que tenemos dos cuentas:

A = $100
B = $100

Y una transacción T1 que quiere transferir $50 de A a B:

T1:
1. A = A - 50       → A = 50
2. B = B + 50       → B = 150
3. COMMIT

La atomicidad se preserva: T1 realiza sus dos operaciones o, si falla antes del COMMIT, se deshacen ambas.

Ahora supongamos que concurrentemente ejecutamos T2, que consulta el saldo de A entre los pasos 1 y 2:

T1: A = A - 50      → A = 50
T2: lee A           → 50
T1: B = B + 50      → B = 150
T1: COMMIT

T2 pudo observar un estado intermedio de T1. Por lo tanto, el aislamiento no se preserva, aunque T1 siga siendo atómica.

Al revés: aislamiento preservado, atomicidad no

Imaginemos que T1 hace:

T1:
1. A = A - 50       → A = 50
2. falla el sistema
3. B = B + 50       ❌ nunca se ejecuta

Si el sistema garantiza aislamiento, otra transacción no verá necesariamente ese estado intermedio de T1.

Pero si no hay atomicidad, puede quedar:

A = 50
B = 100

La transferencia quedó a medias.

Atomicidad evita que una transacción quede parcialmente ejecutada (todo o nada), mientras que aislamiento evita que una transacción observe o interfiera incorrectamente con los estados intermedios de otras transacciones concurrentes.

Atomicidad → problema de ejecución incompleta.
Aislamiento → problema de concurrencia.

3) Hay que mirar operaciones de distintas trnsacciones que accedan al msimo dato
T1: R(A)        T2: W(A)
| Operación de T1 | Operación de T2 | Conflicto          | Ejemplo             |
| --------------- | --------------- | ------------------ | ------------------- |
| **R**           | **W**           | **RW**             | T1: R(A) → T2: W(A) |
| **W**           | **R**           | **WR**             | T1: W(A) → T2: R(A) |
| **W**           | **W**           | **WW**             | T1: W(A) → T2: W(A) |
| **R**           | **R**           | ❌ No hay conflicto | T1: R(A) → T2: R(A) |

Regla rápida

Para cada par de operaciones:

¿Son de transacciones diferentes?
¿Acceden al mismo dato?
¿Al menos una escribe?

Si las tres respuestas son sí, hay conflicto.

Y el tipo se determina por el orden:

R → W = RW
W → R = WR
W → W = WW
R → R = nada

4) Para construir el grafo de precedencia de un schedule, se transforman los conflictos enter transacciones en aristas dirigidas. 

Ejemplo

Supongamos este schedule:

R1(A)
W2(A)
W1(B)
R2(B)

Tenemos dos transacciones: T1 y T2.

1. Buscamos conflictos

R1(A) → W2(A)
Es un conflicto RW. Como T1 ocurre antes que T2:

T1 → T2

W1(B) → R2(B)
Es un conflicto WR. Como T1 ocurre antes que T2:

T1 → T2

No hace falta agregar dos veces la misma arista.

2. Construimos el grafo
T1 ─────→ T2
Nodos: una por cada transacción.
Aristas: una por cada dependencia causada por un conflicto.
3. ¿Es conflict-serializable?

La regla fundamental es:

Un schedule es conflict-serializable si y solo si su grafo de precedencia es acíclico.

En este caso:

T1 → T2

No hay ningún ciclo, por lo tanto:

✅ El schedule es conflict-serializable.

Además, el orden serial equivalente es:

T1 → T2
Ejemplo donde NO es serializable

Consideremos:

R1(A)
W2(A)
R2(B)
W1(B)

Los conflictos son:

R1(A) → W2(A) → T1 → T2
R2(B) → W1(B) → T2 → T1

El grafo queda:

      ┌──────→ T2
      │         │
      │         ↓
     T1 ←───────┘

Es decir:

T1 → T2
T2 → T1

Hay un ciclo:

T1 → T2 → T1

Por lo tanto:

❌ No es conflict-serializable.

Para resolver cualquier ejercicio

Podés seguir siempre estos 4 pasos:

Identificar las transacciones → nodos.
Buscar conflictos RW, WR y WW entre transacciones diferentes.
Agregar una arista Ti → Tj si la operación de Ti ocurre antes que la conflictiva de Tj.
Buscar ciclos:
Sin ciclo → ✅ conflict-serializable
Con ciclo → ❌ no conflict-serializable

R-R nunca genera una arista, porque dos lecturas no generan conflicto.

5) Orden serial equivalente: orden en que ejecuto las transacciones una después de la otra, sin intercalarlas, y que produce el mismo comportamiento relevante que el schedule original.

Ejemplo

Tenemos este schedule intercalado:

R1(A)
W1(A)
R2(B)
W2(B)
R1(C)
W2(C)

Las operaciones de T1 y T2 están mezcladas.

Un posible orden serial sería:

T1:
R1(A)
W1(A)
R1(C)

T2:
R2(B)
W2(B)
W2(C)

Es decir:

$$ \boxed{T1 \rightarrow T2} $$

Eso es serial porque primero termina completamente T1 y después empieza T2.

¿Cuándo es "equivalente"?

No alcanza con que tenga las mismas transacciones. El orden serial debe respetar los conflictos del schedule original.

Por ejemplo, si en el schedule original tenemos:

W1(A)
R2(A)

hay un conflicto WR, y obliga a:

$$ T1 \rightarrow T2 $$

Por lo tanto, un orden serial equivalente no podría ser:

$$ T2 \rightarrow T1 $$

porque estaría invirtiendo ese conflicto.

Relación con el grafo

Esta es justamente la utilidad del grafo de precedencia:

Sin ciclos → podemos encontrar un orden de las transacciones que respete todas las aristas → existe un orden serial equivalente.
Con ciclo → las restricciones se contradicen → no existe orden serial equivalente.

----------------------------------------------------------------------------------------------------------

La idea clave es que un ciclo impone requisitos de orden contradictorios.

Supongamos que el grafo tiene un ciclo:

$$ T_1 \rightarrow T_2 \rightarrow T_3 \rightarrow T_1 $$

Esto significa que, debido a conflictos del schedule original:

\(T_1\) debe ir antes que \(T_2\)
\(T_2\) debe ir antes que \(T_3\)
\(T_3\) debe ir antes que \(T_1\)

Entonces, cualquier orden serial equivalente tendría que cumplir:

$$ T_1 < T_2 < T_3 < T_1 $$

Pero esto es imposible: estaríamos exigiendo que \(T_1\) sea a la vez anterior y posterior a sí misma.

Por lo tanto, no existe ningún ordenamiento serial de las transacciones que pueda respetar todos los conflictos del schedule original.

Un ciclo en el grafo implica una cadena de dependencias que exige que una transacción preceda a otra, y que finalmente esta última preceda nuevamente a la primera. Como un orden serial debe ser lineal y no puede contener ciclos, no existe un orden serial equivalente. Por eso, un schedule con un ciclo no es conflict-serializable.

## Nivel 2: ejecuciones y fallas

1) Un dirty read ocurre cuando una transacción lee un dato que fue modificado por otra trnasacción, pero esa n¿modiciación todavía no fue confirmada con un COMMIT. 

Un dirty read ocurre cuando una transacción lee una modificación no confirmada de otra transacción. Si la transacción que realizó la modificación hace ABORT, el valor leído desaparece mediante el rollback, por lo que la segunda transacción había observado un estado que finalmente no pertenece al estado válido de la base de datos.

A=100

El schedule es:

T1: W(A = 50)
T2: R(A) → 50
T1: ABORT

Paso a paso:

T1 modifica A de 100 a 50.
T1 todavía no hizo COMMIT.
T2 lee A y obtiene 50.
T1 hace ABORT, por lo que su modificación se deshace:
$$ A \leftarrow 100 $$
¿Dónde está el problema?

T2 observó:

$$ A = 50 $$

pero ese valor nunca fue confirmado. Después del ABORT de T1, el valor válido vuelve a ser:

$$ A = 100 $$

Por eso se llama dirty read (lectura sucia): T2 leyó un dato que posteriormente fue deshecho.

Estado observable después del ABORT

Después de:

W1(A=50)
R2(A)       → T2 observa 50
ABORT1

el estado de la base queda:

A = 100

pero T2 ya había observado 50. Es decir, T2 pudo tomar decisiones basándose en un valor que finalmente no existió de manera válida.

2) Non-repeatable read ocurre cuando una transacción lee dos veces el mismo dato y obtiene valores diferentes, porque otra transacción lo modificó y confirmó entre ambas lecturas. 
![img.png](images/img.png)
Una non-repeatable read ocurre cuando una transacción lee dos veces el mismo dato y obtiene valores diferentes debido a una actualización confirmada por otra transacción entre ambas lecturas. Es posible en READ COMMITTED, pero no en REPEATABLE READ ni en SERIALIZABLE.
![img_1.png](images/img_1.png)
|                                   | Dirty read                     | Non-repeatable read                    |
| --------------------------------- | ------------------------------ | -------------------------------------- |
| ¿Cuántas veces lee T1?            | Una puede alcanzar             | **Dos**                                |
| ¿La otra transacción hizo COMMIT? | ❌ No                           | ✅ Sí                                   |
| ¿Puede haber ABORT?               | **Sí, es lo característico**   | No es necesario                        |
| Problema                          | Leí un valor **no confirmado** | El valor **cambió entre mis lecturas** |

Truquito para acordarte

Dirty:

"Leí algo sucio, que todavía no estaba confirmado."

Non-repeatable:

"Lo leí dos veces, pero no pude repetir el resultado."

Por ejemplo, pensalo como el saldo de una cuenta:

Dirty read:

"Vi $200, pero el cambio se canceló. En realidad nunca quedaron $200."

Non-repeatable read:

"Vi $100. Volví a mirar y ahora hay $200 porque alguien hizo una transferencia confirmada en el medio."

Dirty read: el problema es que se toma una decisión basándose en un dato que finalmente no existió
Non-repeatable read: el probelma es que no se puede confirmar que el dato que leyó siga siendo igual durante su transacción

3) (Predicado: condición que ponés en la consulta para decidir qué filas entran en el reultado. SELECT * FROM empleados WHERE salario > 1100; el predicado es salario>1100)
Un phantom read ocurre cuando una transacción ejecuta dos veces la misma consulta por predicado y, entre ambas consultas, otra transacción inserta, elimina o modifica una fila de manera que pasa a cumplir el predicado.

Un phantom read ocurre cuando una transacción ejecuta dos veces una consulta sobre un predicado y obtiene conjuntos de resultados diferentes porque otra transacción insertó, eliminó o modificó filas que cumplen dicho predicado entre ambas consultas.
En una base de datos real es totalmente esperable que se agreguen o eliminen datos mientras vos trabajás. El punto del phantom read no es que eso sea "incorrecto".

El problema aparece cuando tu transacción necesita trabajar como si estuviera viendo un conjunto estable de datos.

¿Por qué puede ser un problema?

Porque T1 podría estar haciendo algo que requiere que ese conjunto no cambie mientras dura su operación. Esto estaría sucediendo dentro e la msima transacción. 

| id | nombre | salario |
| -- | ------ | ------: |
| 1  | Ana    |    1000 |
| 2  | Juan   |    1200 |

T1 hace una consulta:

SELECT *
FROM Empleados
WHERE salario > 1100;

Obtiene:

Juan | 1200

Ahora T2 inserta un nuevo empleado:

INSERT INTO Empleados VALUES (3, 'Pedro', 1500);
COMMIT;

T1 vuelve a ejecutar exactamente la misma consulta:

SELECT *
FROM Empleados
WHERE salario > 1100;

Ahora obtiene:

Juan  | 1200
Pedro | 1500
¿Qué apareció?

Pedro es el "fantasma". 👻

No estaba en el resultado de la primera consulta, pero apareció en la segunda porque otra transacción insertó una fila que cumple el mismo predicado.

Diferencia con non-repeatable read

Esta distinción es importante:

Non-repeatable read:

La misma fila existe, pero cambia su valor.

T1: SELECT salario WHERE id=2 → 1200
T2: UPDATE ... SET salario=1500
T2: COMMIT
T1: SELECT salario WHERE id=2 → 1500

Phantom read:

Aparece o desaparece una fila completa del resultado de una consulta por predicado.

T1: SELECT * WHERE salario > 1100 → {Juan}
T2: INSERT Pedro (1500)
T2: COMMIT
T1: SELECT * WHERE salario > 1100 → {Juan, Pedro}

4) Write skew. Hay dos médicos de guardia. Sólo se pueden ir si queda otro. 
M1 mira y está M2 
M2 mira y está M1
Ambos deciden irse, ya no hay médicos!

Lo importante del write skew

No es que cada transacción haya hecho algo incorrecto individualmente:

T1: "Si M2 sigue, yo puedo irme." ✅
T2: "Si M1 sigue, yo puedo irme." ✅

El problema aparece por la combinación de ambas decisiones:

Cada una verifica el invariante usando un estado que era válido, pero las dos actualizaciones juntas hacen que el invariante deje de cumplirse.

Invariante: debe haber al menos un médico de guardia, es decir, \(M_1 + M_2 \geq 1\). El write skew ocurre porque ambas transacciones leen un estado en el que el invariante se cumple y, de forma concurrente, realizan actualizaciones que individualmente parecen válidas pero conjuntamente lo violan, dejando \(M_1+M_2=0\).

5) La clave está en que Snapshot Isolation controla qué versiones de los datos ve cada transacción, y además evita que dos transacciones hagan write-write sobre la misma fila al mismo tiempo. Pero eso no alcanza para evitar write skew, porque en el write skew cada transacción escribe una fila distinta.

1. ¿Por qué evita WW?

En Snapshot Isolation, cada transacción trabaja sobre un snapshot consistente.

Si dos transacciones intentan modificar la misma fila, por ejemplo:

T1: W(A)
T2: W(A)

hay un conflicto WW.

Snapshot Isolation detecta que ambas quieren escribir sobre A y no permite que las dos confirmen. Una debe abortar.

Por eso:

$$ \boxed{\text{SI evita conflictos WW}} $$
2. ¿Pero por qué permite write skew?

Volvamos a los médicos:

M1 = de guardia
M2 = de guardia

Tenemos:

$$ M_1 + M_2 \geq 1 $$

Ahora:

T1 (M1):
    lee M2 = de guardia
    → decide irse
    W(M1 = fuera)

T2 (M2):
    lee M1 = de guardia
    → decide irse
    W(M2 = fuera)

Fijate en algo fundamental:

T1 escribe M1
T2 escribe M2

No escriben la misma fila.

Por lo tanto, no hay WW:

$$ W_1(M_1) \quad\text{y}\quad W_2(M_2) $$

Como Snapshot Isolation principalmente detecta el conflicto cuando las dos quieren escribir el mismo dato, permite que ambas hagan COMMIT.

Resultado:

M1 = fuera
M2 = fuera

y:

$$ M_1 + M_2 = 0 $$

❌ Se viola el invariante.

La idea para memorizar

WW:

"Los dos quieren modificar el mismo dato."
→ SI lo detecta → uno aborta.

Write skew:

"Los dos modifican datos distintos, pero sus modificaciones juntas rompen una regla."
→ SI puede no detectarlo → ambos confirman.

Por eso:

$$ \boxed{\text{Snapshot Isolation evita WW, pero no garantiza la ausencia de write skew}} $$

Y esta es justamente una de las diferencias importantes entre Snapshot Isolation y Serializable: Serializable debe impedir también estas dependencias que, aunque no sean WW directos, producen un resultado no serializable.

## Nivel 3: diseño y comparación

1) Cuanto más fuerte es el nivel, menos anomalías permite. 
| Nivel                  | Dirty read | Non-repeatable read |                       Phantom read | Write skew |
| ---------------------- | ---------: | ------------------: | ---------------------------------: | ---------: |
| **Read Committed**     |          ❌ |                   ✅ |                                  ✅ |          ✅ |
| **Repeatable Read**    |          ❌ |                   ❌ |    ⚠️ depende de la implementación |          ✅ |
| **Snapshot Isolation** |          ❌ |                   ❌ | ❌ en el snapshot de la transacción |          ✅ |
| **Serializable**       |          ❌ |                   ❌ |                                  ❌ |          ❌ |

1. Read Committed

Solo permite leer datos que ya fueron confirmados.

Por eso:

❌ Dirty read: no podés leer un cambio no confirmado.
✅ Non-repeatable read: otra transacción puede modificar y confirmar el dato entre tus dos lecturas.
✅ Phantom read: otra transacción puede insertar/eliminar filas que cumplen tu WHERE.
✅ Write skew: las transacciones pueden leer un estado consistente y modificar filas distintas de forma que rompan un invariante.
2. Repeatable Read

Garantiza que si leíste una fila, al volver a leerla durante la transacción no vas a obtener un valor diferente.

Por eso:

❌ Dirty read
❌ Non-repeatable read
⚠️ Phantom read: depende de cómo esté implementado el nivel en el DBMS. En algunos sistemas puede ocurrir.
✅ Write skew: puede ocurrir porque dos transacciones pueden leer datos consistentes y escribir filas diferentes.
3. Snapshot Isolation

Cada transacción trabaja sobre un snapshot consistente de la base.

Por eso no ve los cambios que otras transacciones confirman después de que comenzó.

❌ Dirty read
❌ Non-repeatable read
❌ Phantom read dentro del snapshot
✅ Write skew

Y acá está la parte importante que vimos antes:

Snapshot Isolation evita WW, pero write skew no necesita WW.

Ejemplo:

T1 lee M2 → está de guardia
T2 lee M1 → está de guardia

T1 escribe M1 → se va
T2 escribe M2 → se va

Escriben filas diferentes, así que SI puede permitir que ambas hagan COMMIT.

4. Serializable

Es el nivel más fuerte.

La ejecución concurrente debe producir un resultado equivalente a algún orden serial.

Por lo tanto, no permite las anomalías anteriores:

❌ Dirty read
❌ Non-repeatable read
❌ Phantom read
❌ Write skew

En particular, en el caso de los médicos, el sistema tendría que impedir que ambas transacciones confirmen simultáneamente porque el resultado no sería equivalente a ninguna ejecución serial.

Read Committed
    ↓
"Solo veo COMMIT"
    ↓
Repeatable Read
    ↓
"Lo que leí no cambia"
    ↓
Snapshot Isolation
    ↓
"Veo un snapshot fijo"
    ↓
Serializable
    ↓
"Es como si fueran una por una"

Snapshot Isolation es muy fuerte contra lecturas inconsistentes y conflictos WW, pero no garantiza serializabilidad porque puede permitir write skew. Serializable sí evita write skew.

2) La prueba para diferenciar snapshot isolation de serializabilidad real es write skew porque SI lo puede permitir meitnras que serializable real no. 
En el caso de los dos méricos, la invariante es M1 + M2 >= 1. Si ejecutoen simultáneo dos transacciones:

T1 (M1):                 T2 (M2):

R(M2) → de guardia       R(M1) → de guardia
W(M1) → fuera            W(M2) → fuera
COMMIT                   COMMIT

![img.png](images/img.png)
![img_1.png](images/img_1.png)
![img_2.png](images/img_2.png)

SI detecta los WW, la clave está en construir un caso donde cada trnsacción escriba algo distinto pero las dos juntas rompan una regla.

Para diferenciar Snapshot Isolation de serializabilidad, se puede utilizar una prueba de write skew. Dos transacciones leen un snapshot consistente, modifican filas diferentes y sus modificaciones conjuntas violan un invariante. Snapshot Isolation puede permitir que ambas hagan COMMIT porque no existe un conflicto WW, mientras que un sistema Serializable debe impedir esa ejecución, ya que no es equivalente a ningún orden serial.

3) El protocolo **Two-Phase Commit (2PC)** coordina una transacción distribuida $T$ entre un **Coordinador ($C$)** y tres **Participantes ($P_1, P_2, P_3$)** para garantizar atomicidad global (todo o nada). 

Para sobrevivir a caídas y permitir la recuperación consistente sin perder el estado, cada nodo escribe registros en su **almacenamiento persistente / WAL (Write-Ahead Log)** antes de emitir mensajes clave a la red.

Fase 1: Preparación (Prepare / Voting Phase)

1. El Coordinador decide iniciar el commit de la transacción:
   * Escribe en su log durable: `START_2PC(T)`.
   * Envía el mensaje `PREPARE` a los participantes $P_1, P_2, P_3$.
2. Cada participante $P_i$ recibe `PREPARE` y evalúa localmente si puede confirmar sus cambios:
   * Verifica restricciones de integridad, bloquea los recursos involucrados (locks) y asegura que puede persistir el cambio.
   * **Si puede confirmar:**
     * Escribe en su log durable con sincronización a disco (*fsync*): `PREPARED(T)`.
     * Envía `VOTE_COMMIT` (o `YES`) al Coordinador.
     * *Promesa:* A partir de este momento, $P_i$ garantiza que podrá hacer commit pase lo que pase, y retiene todos los locks hasta recibir la decisión final.
   * **Si no puede confirmar:**
     * Escribe en su log durable: `ABORT(T)`.
     * Envía `VOTE_ABORT` (o `NO`) al Coordinador y aborta localmente liberando recursos.

Fase 2: Decisión y Compromiso (Commit / Abort Phase)

1. **Decisión del Coordinador:**
   * **Caso Éxito Unánime:** Si recibe `VOTE_COMMIT` de los tres participantes ($P_1, P_2, P_3$):
     * Escribe en su log durable con *fsync*: `GLOBAL_COMMIT(T)`. *(Punto formal de no retorno: la transacción se considera confirmada)*.
     * Envía el mensaje `GLOBAL_COMMIT` a $P_1, P_2, P_3$.
   * **Caso Falla o Timeout:** Si al menos un participante responde `VOTE_ABORT`, o si vence un timeout esperando los votos:
     * Escribe en su log durable: `GLOBAL_ABORT(T)`.
     * Envía el mensaje `GLOBAL_ABORT` a todos los participantes.
2. **Ejecución en los Participantes:**
   * Al recibir `GLOBAL_COMMIT`:
     * Aplica definitivamente las modificaciones en sus datos.
     * Escribe en su log durable: `COMMIT(T)`.
     * Libera los locks retenidos.
     * Envía un mensaje `ACK` al Coordinador.
   * Al recibir `GLOBAL_ABORT`:
     * Realiza rollback de las modificaciones locales.
     * Escribe en su log durable: `ABORT(T)`.
     * Libera los locks retenidos.
     * Envía un mensaje `ACK` al Coordinador.
3. **Cierre de la transacción:**
   * Al recibir los tres `ACK`s de $P_1, P_2, P_3$, el Coordinador escribe en su log: `END(T)`. A partir de allí, la información de $T$ en el log puede archivarse o recolectarse.

```
COORDINADOR                    P1             P2             P3
    │                          │              │              │
[WAL: START_2PC]               │              │              │
    ├───── PREPARE ───────────►│              │              │
    ├───── PREPARE ──────────────────────────►│              │
    ├───── PREPARE ─────────────────────────────────────────►│
    │                          │              │              │
    │                    [WAL: PREPARED] [WAL: PREPARED] [WAL: PREPARED]
    │◄──── VOTE_COMMIT (YES) ──┤              │              │
    │◄──── VOTE_COMMIT (YES) ─────────────────┤              │
    │◄──── VOTE_COMMIT (YES) ────────────────────────────────┤
    │                          │              │              │
[WAL: GLOBAL_COMMIT] (fsync)   │              │              │
    ├───── GLOBAL_COMMIT ─────►│              │              │
    ├───── GLOBAL_COMMIT ────────────────────►│              │
    ├───── GLOBAL_COMMIT ───────────────────────────────────►│
    │                          │              │              │
    │                    [WAL: COMMIT]  [WAL: COMMIT]  [WAL: COMMIT]
    │                     (libera locks) (libera locks) (libera locks)
    │◄──── ACK ────────────────┤              │              │
    │◄──── ACK ───────────────────────────────┤              │
    │◄──── ACK ──────────────────────────────────────────────┤
    │                          │              │              │
[WAL: END]                     │              │              │
```

| Entidad | Momento | Registro en WAL | ¿Requiere fsync forzado? | Propósito |
| :--- | :--- | :--- | :---: | :--- |
| **Coordinador** | Antes de enviar `PREPARE` | `START_2PC(T)` | No estrictamente | Registra el inicio del protocolo. |
| **Participante** | Antes de responder `YES` | `PREPARED(T)` | **Sí** | Promete irrevocablemente poder comitear y retiene locks. |
| **Participante** | Antes de responder `NO` | `ABORT(T)` | Sí | Cancela la transacción localmente de forma segura. |
| **Coordinador** | Tras recibir todos los `YES` | `GLOBAL_COMMIT(T)` | **Sí** | Punto de compromiso oficial; la transacción ya es irreversible. |
| **Coordinador** | Si hay un voto `NO` o timeout | `GLOBAL_ABORT(T)` | Sí | Cancela la transacción globalmente. |
| **Participante** | Tras recibir la decisión | `COMMIT(T)` / `ABORT(T)` | Sí | Registra la finalización local y libera locks. |
| **Coordinador** | Tras recibir todos los `ACK` | `END(T)` | No estrictamente | Cierra la transacción en el coordinador. |

4) Análisis de una caída del coordinador en los tres momentos del protocolo:

1. **Antes de `PREPARE`:**
   * **Situación:** El coordinador se cae antes de registrar `START_2PC` o antes de enviar mensajes `PREPARE` a los participantes.
   * **Comportamiento de los participantes:** Los participantes no recibieron ninguna solicitud de preparación. Si mantuvieron bloqueos temporales por operaciones preliminares de la transacción, al vencer sus timeouts de inactividad asumen falla del coordinador y deciden unilateralmente hacer `ABORT` local y liberar recursos.
   * **Recuperación del coordinador:** Al reiniciar, el coordinador revisa su log; como no hay constancia de inicio de 2PC, la transacción simplemente se descarta o se aborta si el cliente reintenta. La atomicidad se preserva sin conflicto porque ningún nodo comiteó.

2. **Durante `PREPARE`:**
   * **Situación:** El coordinador escribió `START_2PC` y envió `PREPARE` a algunos o todos los participantes, pero cae antes de recibir todos los votos y antes de persistir una decisión (`GLOBAL_COMMIT` o `GLOBAL_ABORT`).
   * **Comportamiento de los participantes:**
     * Cualquier participante que haya votado `NO` ya abortó localmente de forma definitiva.
     * Aquellos participantes que recibieron `PREPARE` y votaron `YES` están en estado `PREPARED`. Al vencer su timeout sin respuesta del coordinador, entran en **incertidumbre (bloqueo)**: no saben si otros participantes votaron `NO` o si el coordinador llegó a decidir algo antes de caer. Deben permanecer a la espera reteniendo todos los locks para no romper la atomicidad.
   * **Recuperación del coordinador:** Al volver a levantarse, el coordinador lee su WAL y encuentra `START_2PC(T)` pero **ningún registro de decisión**. La regla de recuperación segura es que, ante la falta de un commit confirmado, se asume abort:
     * Escribe `GLOBAL_ABORT(T)` en su WAL.
     * Envía `GLOBAL_ABORT` a todos los participantes.
     * Los participantes que estaban en `PREPARED` reciben la orden, deshacen sus cambios, liberan sus locks y el sistema se desbloquea de forma consistente.

3. **Después de recibir todos los votos YES:**
   Acá es fundamental distinguir si la decisión llegó a escribirse en el disco antes de la caída:
   * **Subcaso A (Cae antes de escribir en disco):** Recibió todos los `YES` en memoria, pero el servidor cayó antes de completar el *fsync* de `GLOBAL_COMMIT` en el WAL.
     * Al reiniciar, el log no contiene la decisión de commit. Siguiendo la regla de seguridad, el coordinador debe registrar `GLOBAL_ABORT(T)` y enviar abort a todos los participantes. Como ninguno había comiteado, la atomicidad se preserva.
   * **Subcaso B (Cae después de escribir `GLOBAL_COMMIT` en el WAL):** Logró persistir `GLOBAL_COMMIT(T)` en disco, pero cayó antes de enviar el mensaje a los participantes (o mientras lo enviaba).
     * **Estado de los participantes:** $P_1, P_2, P_3$ están en estado `PREPARED`, bloqueados a la espera de la decisión oficial.
     * **Recuperación del coordinador:** Al reiniciarse, el coordinador lee su WAL y encuentra `GLOBAL_COMMIT(T)`. Sabe con certeza que la transacción fue formalmente confirmada. Inmediatamente retransmite `GLOBAL_COMMIT` a todos los participantes. Los participantes aplican el commit, liberan locks y responden `ACK`.

La conclusión principal es que **2PC es un protocolo bloqueante**: si el coordinador cae mientras los participantes están en estado `PREPARED`, estos no pueden resolver la transacción por su cuenta y deben mantener sus recursos bloqueados hasta que el coordinador se recupere.

5) Un participante en estado `PREPARED` no puede decidir unilateralmente porque **carece de información sobre el estado global del sistema** y cualquier decisión individual que tome introduce un riesgo crítico de violar la **Atomicidad** ("todo o nada"):

1. **Riesgo si decidiera unilateralmente hacer ABORT:**
   * Supongamos que el participante $P$ espera al coordinador, se vence su timeout y piensa: *"Como no responde, aborto y libero mis locks"*.
   * **Peligro real:** Puede haber ocurrido que todos los participantes (incluido $P$) hayan respondido `VOTE_COMMIT`, el coordinador haya recolectado todos los votos, haya persistido `GLOBAL_COMMIT` en su WAL y le haya llegado a notificar el commit a los demás participantes justo antes de caerse o aislarse de la red.
   * **Consecuencia:** Los otros participantes comitearon definitivamente, mientras que $P$ abortó. Se produjo una confirmación parcial de la transacción, rompiendo la atomicidad.

2. **Riesgo si decidiera unilateralmente hacer COMMIT:**
   * Supongamos que el participante $P$ piensa: *"Como yo voté YES y estoy listo, asumo que todo salió bien y confirmo mis cambios"*.
   * **Peligro real:** Otro de los participantes pudo haber votado `VOTE_ABORT` (por ejemplo, por falta de saldo, violación de integridad o falla de hardware), o un timeout previo hizo que el coordinador decidiera y persistiera `GLOBAL_ABORT`.
   * **Consecuencia:** $P$ confirmó los cambios mientras que los demás los descartaron. Nuevamente se viola la atomicidad.

La idea clave:

> Al emitir el voto `VOTE_COMMIT` y escribir `PREPARED` en el log, el participante **renuncia voluntariamente a su autonomía**. Transfiere la autoridad de decisión al coordinador. Queda en una "zona de incertidumbre": no puede confirmar porque otro pudo haber votado NO, y no puede cancelar porque el sistema pudo haber decidido COMMIT. Por lo tanto, está obligado a esperar bloqueado hasta recibir la decisión oficial.

## Nivel 4: integración y defensa

1) **2PC vs. Consenso (Paxos / Raft)**

Aunque ambos son mecanismos fundamentales de coordinación distribuida, resuelven problemas conceptualmente diferentes:

* **Problema que resuelve cada uno:**
  * **2PC (Compromiso Atómico / Atomic Commitment):** Resuelve el acuerdo sobre la **confirmación atómica de transacciones distribuidas** donde los datos están particionados. La pregunta es: *"¿Todos y cada uno de los participantes pueden aplicar su parte de la transacción?"*. Requiere **unanimidad**: si un participante no puede, toda la transacción debe cancelarse.
  * **Consenso (Paxos / Raft):** Resuelve el acuerdo sobre un **único valor o secuencia de comandos en un log replicado**. La pregunta es: *"¿Podemos ponernos de acuerdo en el próximo estado a pesar de que algunos nodos fallen?"*. Requiere **quórum mayoritario ($\lfloor n/2 \rfloor + 1$)**: los nodos son réplicas redundantes del mismo estado.

* **Seguridad (Safety):**
  * En **2PC**: Garantiza que nunca ocurrirá una ejecución parcial. Todos comitean o todos abortan. No se permite confirmación si algún participante votó abort o falló.
  * En **Consenso**: Garantiza acuerdo (nunca se eligen dos valores distintos para la misma posición del log) y validez (el valor elegido fue propuesto por un nodo).

* **Progreso (Liveness):**
  * En **2PC**: **Es bloqueante**. Si el coordinador se cae mientras los participantes están en `PREPARED`, o si un nodo falla durante la fase de votación, el protocolo no puede avanzar. La disponibilidad se sacrifica totalmente para preservar la consistencia.
  * En **Consenso**: **No es bloqueante ante caídas minoritarias**. En un clúster de $2f + 1$ nodos, puede tolerar la caída o desconexión de hasta $f$ nodos sin detenerse. Mientras una mayoría esté viva y comunicada, el sistema sigue procesando operaciones y comiteando.

| Criterio | Two-Phase Commit (2PC) | Consenso (Paxos / Raft) |
| :--- | :--- | :--- |
| **Problema central** | Atomic Commitment (datos particionados) | State Machine Replication (datos replicados) |
| **Condición de éxito** | **Unanimidad (100% de los votos)** | **Mayoría / Quórum ($> 50\%$)** |
| **Tolerancia a fallas** | 0 fallas toleradas en votación (1 fallo detiene todo) | Tolera $f$ caídas en un grupo de $2f+1$ |
| **Progreso (Liveness)** | Bloqueante ante caídas del coordinador | No bloqueante mientras exista quórum activo |
| **Rol de los nodos** | Participantes heterogéneos con datos distintos | Réplicas homogéneas con copias del mismo dato |

2) **Three-Phase Commit (3PC) y las particiones de red**

3PC fue diseñado como una extensión de 2PC con el objetivo de ser un protocolo de compromiso **no bloqueante (non-blocking)**. Para lograrlo, divide la fase de decisión incorporando un estado intermedio: `CanCommit?` $\rightarrow$ `PreCommit` $\rightarrow$ `DoCommit`. De esta forma, ningún participante puede comitear mientras otro permanezca en la fase inicial de votación, permitiendo que ante la caída del coordinador los participantes puedan deducir el estado y resolver la transacción cooperativamente.

* **Supuesto adicional que necesita 3PC:**
  * 3PC requiere un **detector de fallas perfecto (Perfect Failure Detector)** en un modelo de **red síncrona**.
  * Asume que existen límites máximos conocidos y estrictos de retardo de red ($\Delta$) y de velocidad de procesamiento.
  * Con este supuesto, si expira un temporizador sin recibir mensaje de un nodo, el protocolo concluye con certeza absoluta que el nodo sufrió un *crash* (murió). No existe la posibilidad de que el mensaje esté simplemente retrasado.

* **Por qué NO elimina las particiones reales:**
  * En redes reales (asíncronas o parcialmente síncronas, como Internet o redes entre datacenters), **las particiones de red son indistinguibles de una caída de nodos**.
  * Si la red se parte en dos subgrupos aislados:
    * El subgrupo donde se encuentran los nodos que alcanzaron el estado `PreCommit` detecta timeout del coordinador, asume que cayó y decide avanzar cooperativamente a `DoCommit`.
    * El otro subgrupo particionado, que quedó en `CanCommit` sin recibir el `PreCommit`, detecta timeout, asume caída del coordinador y decide abortar (`DoAbort`).
    * **Resultado:** Se produce una situación de **Split-Brain**: un grupo confirma y el otro cancela, destruyendo por completo la atomicidad.
  * Por el teorema FLP y los límites fundamentales del compromiso distribuido, ningún protocolo puede ser simultáneamente no bloqueante y tolerante a particiones de red arbitrarias sin basarse en consensos de quórum mayoritario. Por esta razón, 3PC tiene nula aplicación práctica en la industria.

3) **Diseño de una Saga: Reserva, Pago y Emisión**

Una Saga descompone una transacción distribuida en una secuencia de transacciones locales ($T_1, T_2, T_3$), donde cada servicio confirma sus cambios en su propia base de datos. Si un paso falla, se ejecutan transacciones compensatorias ($C_2, C_1$) en orden inverso para revertir semánticamente las operaciones previas.

Estructura de la Saga (mediante un **Orquestador de Sagas**):

| Paso ($i$) | Transacción Local ($T_i$) | Acción local | Transacción Compensatoria ($C_i$) | Acción de reversión |
| :---: | :--- | :--- | :--- | :--- |
| **1** | $T_1$: Reservar asiento | Bloquea el asiento en el vuelo (`PENDING`) | $C_1$: Cancelar reserva | Libera el asiento para otros usuarios |
| **2** | $T_2$: Cobrar pasaje | Captura y debita el importe en la tarjeta | $C_2$: Reembolsar dinero | Emite un refund del importe cobrado |
| **3** | $T_3$: Emitir ticket | Genera código de ticket y factura fiscal | $C_3$: Anular ticket | Revoca el ticket emitido *(pivote)* |

* **Camino Feliz:**
  $$ T_1 \ (\text{Reserva}) \longrightarrow T_2 \ (\text{Pago}) \longrightarrow T_3 \ (\text{Emisión}) \longrightarrow \text{Confirmado} $$
  Cada servicio responde exitosamente al orquestador y la saga concluye con éxito.

* **Flujo con Falla y Compensación:**
  Supongamos que $T_1$ y $T_2$ se ejecutaron con éxito, pero $T_3$ falla (por ejemplo, el sistema de la aerolínea no responde o agotó el cupo de emisión):
  $$ T_1 \longrightarrow T_2 \longrightarrow T_3 \ (\text{ERROR}) \Longrightarrow C_2 \ (\text{Reembolsar}) \longrightarrow C_1 \ (\text{Liberar asiento}) $$
  1. El orquestador detecta el fallo irreversible en $T_3$.
  2. Invoca $C_2$: el servicio de pagos realiza el reembolso del dinero cobrado.
  3. Invoca $C_1$: el servicio de reservas libera el asiento bloqueado.
  4. La saga finaliza en estado `CANCELLED`, dejando el sistema en un estado de negocio consistente.

* **Idempotencia en la Saga:**
  En una red distribuida, los mensajes de solicitud o reintento pueden duplicarse o llegar con retraso.
  * Cada saga se crea con un identificador unívoco global (`saga_id`).
  * Cada paso genera una **Idempotency Key** derivada (ej.: `saga_id + "-payment"`).
  * **En transacciones normales ($T_i$):** Si se produce un timeout en $T_2$ y el orquestador reintenta el cobro, el servicio de pagos consulta su registro de transacciones procesadas con esa clave; si ya fue cobrado, responde con el comprobante anterior en vez de cobrar dos veces.
  * **En transacciones compensatorias ($C_i$):** Las compensaciones **deben ser estrictamente idempotentes e infalibles**. Si el reembolso $C_2$ se ejecuta múltiples veces por reintentos de red, el dinero sólo se devuelve una vez y las llamadas duplicadas retornan éxito. Si un servicio compensatorio está caído, el orquestador reintenta con backoff exponencial hasta que la compensación se complete satisfactoriamente.

4) **Aplicación del marco P-F-G-S-T a una transacción distribuida del Trabajo Final**

Caso: **Juego en línea: Mundo virtual medieval** — Transacción distribuida de **Intercambio / Compra-venta de un ítem legendario por monedas de oro entre dos jugadores**.

* **P: problema** $\rightarrow$ Transferencia atómica entre dos servicios independientes (Servicio de Inventario y Servicio de Billetera/Economía). Se debe transferir un ítem único del Jugador A al Jugador B a cambio de 5.000 monedas de oro transferidas de B hacia A. Riesgo de ejecución parcial: que se descuente el dinero sin entregar el ítem, o que se duplique el ítem en ambos inventarios (dupe exploit).
* **F: fallas asumidas** $\rightarrow$ Pérdida de paquetes de red, timeouts en las llamadas RPC entre microservicios, caídas (*crash*) intempestivas del servidor de inventarios o de billeteras, reintentos automáticos enviados por el cliente y desconexiones de red de los jugadores en pleno intercambio.
* **G: garantía requerida** $\rightarrow$ Atomicidad distribuida de negocio: el intercambio es todo o nada. El ítem no puede clonarse ni desaparecer, y el débito/crédito de oro debe ejecutarse exactamente una vez por cada solicitud confirmada.
* **S: solución elegida** $\rightarrow$ Patrón **Saga Orquestada con bloqueo lógico (Escrow / Reserva temporal)** e identificación por `trade_id` único:
  1. *Reserva del ítem:* El servicio de inventario marca el ítem de A en estado `LOCKED_IN_TRADE`.
  2. *Reserva de fondos:* El servicio de billetera retiene las 5.000 monedas de B (pasan a saldo en custodia).
  3. *Transferencia definitiva:* Se asigna el ítem a B y se acreditan las monedas en la cuenta de A.
  Si el paso 2 o 3 falla, se ejecutan las compensaciones: se libera el ítem retenido a A y se reintegran las monedas a B. Cada operación registra su `trade_id` en una tabla de idempotencia para filtrar reintentos duplicados.
* **T: trade-off aceptado** $\rightarrow$ Se prioriza consistencia e integridad frente a disponibilidad inmediata: durante los segundos que dura el intercambio, el ítem y las monedas quedan retenidos sin poder utilizarse en otras acciones del juego. Los jugadores asumen esa breve espera para eliminar cualquier riesgo de duplicación de ítems o pérdida de fondos.

5) **Defensa ante comité: Criterios de elección entre Transacción Fuerte, 2PC y Saga**

Ante un comité técnico de arquitectura, la elección del mecanismo de coordinación para una operación crítica se defiende en función de los límites de los datos, los requerimientos de aislamiento y el impacto en la disponibilidad:

1. **Transacción Fuerte (ACID local en una única base de datos)**
   * **Cuándo elegirla:** Cuando todas las tablas o entidades involucradas en la operación pueden convivir dentro del mismo motor relacional (o cuando el diseño de sharding permite que la transacción ocurra dentro de un único nodo mediante una clave de partición compartida).
   * **Defensa:** Es la solución más simple, robusta y performante. Ofrece garantías matemáticas estrictas de aislamiento (Serializabilidad o Snapshot Isolation) sin overhead de red ni protocolos multipaso, evitando anomalías intermedias sin necesidad de codificar lógica de compensación manual.
   * **Límite:** No escala horizontalmente cuando la operación involucra servicios desacoplados con bases de datos independientes.

2. **Two-Phase Commit (2PC / Atomicidad distribuida estricta)**
   * **Cuándo elegirla:** En operaciones de **muy corta duración dentro de una misma red local de baja latencia y alta confiabilidad**, donde el negocio **prohíbe estrictamente la visibilidad de cualquier estado intermedio incoherente**, ni siquiera por fracciones de segundo (ej.: transferencias contables entre particiones de un core financiero, o motores con soporte nativo de transacciones distribuidas como Google Spanner / CockroachDB).
   * **Defensa:** Asegura consistencia inmediata y aislamiento atómico global: ningún cliente puede observar fondos debitados antes de que se hayan acreditado en el destino.
   * **Trade-off:** Alto acoplamiento temporal y fragilidad en disponibilidad: los recursos quedan bloqueados con locks durante las dos fases. Si el coordinador cae durante la votación, el sistema se bloquea. Resulta inviable para operaciones que involucren APIs externas o redes públicas con latencias impredecibles.

3. **Saga (Consistencia eventual con compensaciones)**
   * **Cuándo elegirla:** En arquitecturas de **microservicios**, procesos de negocio de **larga duración (long-running transactions)**, o cuando la transacción involucra **servicios externos o APIs de terceros** (ej.: procesadores de pago, logística, aerolíneas) donde es técnica o comercialmente imposible retener locks de base de datos.
   * **Defensa:** Maximiza la disponibilidad y escalabilidad horizontal del sistema. Cada servicio realiza commits locales rápidos sin bloquear recursos de otros servicios. Si ocurre un error, el sistema recupera la consistencia mediante transacciones compensatorias.
   * **Trade-off:** Pérdida de aislamiento estricto (los estados intermedios son visibles al sistema) y mayor complejidad de desarrollo (requiere diseñar compensaciones para cada paso y garantizar idempotencia en todos los endpoints).

| Criterio de evaluación | Transacción Fuerte (Local) | 2PC (Distribuido Estricto) | Saga (Consistencia Eventual) |
| :--- | :--- | :--- | :--- |
| **Alcance de los datos** | Misma base de datos | Múltiples nodos / motores homogéneos | Múltiples microservicios / APIs heterogéneas |
| **Aislamiento** | Total (ACID nativo) | Alto (locks distribuidos) | Bajo (estados intermedios visibles) |
| **Disponibilidad** | Máxima localmente | Baja (bloqueante ante caídas) | Muy alta (desacoplada y asíncrona) |
| **Latencia / Throughput** | Óptimo (microsegundos) | Degradado (múltiples viajes de red) | Alto (commits locales asíncronos) |
| **Complejidad de código** | Mínima (provista por el DBMS) | Media (coordinación transaccional) | Alta (orquestación, compensaciones, idempotencia) |

**Regla de oro de defensa:**
> *"Diseñar para **Transacción Fuerte local** siempre que el dominio lo permita. Si la operación involucra múltiples microservicios o dependencias externas, optar por una **Saga con idempotencia y reservas lógicas**. Reservar **2PC** exclusivamente para aquellos escenarios donde el negocio exija consistencia inmediata innegociable y la infraestructura garantice redes locales de latencia mínima y controlada."*