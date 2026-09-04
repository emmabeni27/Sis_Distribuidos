## Nivel 1: fundamentos y reconocimiento

1) **Definición formal del problema de consenso**

En un sistema distribuido compuesto por un conjunto de $n$ procesos $P = \{p_1, p_2, \dots, p_n\}$, el problema de consenso consiste en lograr que los procesos no fallidos se pongan de acuerdo en un único valor común a partir de propuestas individuales.

* **Entradas (Inputs):** Cada proceso $p_i$ comienza con un valor de entrada o propuesta inicial $v_i \in V$ (por ejemplo, en consenso binario $V = \{0, 1\}$; en replicación de logs $V$ representa comandos de una máquina de estados).
* **Salidas (Outputs):** Cada proceso $p_i$ ejecuta una acción irrevocable de decisión $\text{decide}(v)$ donde $v \in V$. La decisión es terminal y definitiva (no se puede cambiar ni retractar).

**Condiciones del sistema (Propiedades requeridas):**
* **Agreement (Acuerdo / Consistencia):** Ningún par de procesos no fallidos decide valores distintos. Si $p_i$ decide $v$ y $p_j$ decide $v'$, entonces $v = v'$.
* **Validity (Validez / No trivialidad):** Si un proceso decide $v$, entonces $v$ debió haber sido propuesto por al menos un proceso del sistema.
* **Termination (Terminación / Liveness):** Todo proceso no fallido eventualmente decide algún valor.
* **Irrevocabilidad:** Una vez que un proceso emite una decisión, el valor queda fijado para siempre y no puede ser alterado.

```
Propuestas iniciales                     Protocolo de Consenso                   Decisión unánime
p1: propone v1 ──────┐
p2: propone v2 ──────┼───────────► [ Intercambio de mensajes ] ──────────► decide(v)
p3: propone v3 ──────┘             (Quórums, Rondas, Votaciones)           (v ∈ {v1, v2, v3})
```

* **Supuestos:** Red asíncrona o parcialmente síncrona, procesos que se ejecutan a velocidades variables y canales de comunicación que pueden sufrir demoras o reordenamientos.
* **Modelo de fallas:** Falla por caída/detención (Crash-stop o Crash-recovery). No se asumen fallas bizantinas (los nodos no mienten ni actúan de manera maliciosa).
* **Garantía:** Se garantiza que el sistema se comporta hacia el exterior como una única entidad coherente (Safety estricto).
* **Trade-off:** La necesidad de coordinar entre múltiples nodos introduce latencia de red obligatoria y mensajes cruzados; además, bajo asincronía total el teorema FLP impone que no se puede asegurar liveness perfecto ante fallas.

---

2) **Propiedades: Agreement, Validity y Termination**

Las propiedades del consenso se dividen fundamentalmente en dos categorías: **Safety** ("nada malo ocurre durante la ejecución") y **Liveness** ("algo bueno eventualmente ocurre").

* **Agreement (Safety):** Garantiza que todos los procesos correctos que deciden tomen exactamente la misma decisión. Evita la divergencia del estado global.
  * *Ejemplo de violación:* En un cluster de base de datos con replicación, ante una partición de red se produce un **Split-Brain**: el nodo A decide aplicar la transacción `x = 10` y el nodo B decide aplicar `x = 20` para la misma operación. Los clientes que leen de A ven un estado incompatible con los que leen de B.

* **Validity (Safety):** Evita soluciones triviales o arbitrarias. Exige que el valor decidido provenga efectivamente de alguna de las propuestas realizadas por los procesos participantes.
  * *Ejemplo de violación:* Los nodos proponen $v_1 = \text{"Transferir \$100"}$ y $v_2 = \text{"Transferir \$200"}$. Sin embargo, el algoritmo tiene un bug o valor por defecto y decide $v = \text{"Transferir \$0"}$, o un valor predefinido que ningún cliente propuso jamás.

* **Termination (Liveness):** Garantiza que el algoritmo no se quede colgado indefinidamente y que todos los nodos no fallidos alcancen una decisión en tiempo finito.
  * *Ejemplo de violación:* Un livelock o bucle infinito de elecciones. Dos candidatos en Raft o Paxos compiten continuamente reiniciando sus temporizadores de forma simétrica; ninguno logra alcanzar la mayoría requerida y el sistema se queda intercambiando mensajes eternamente sin emitir jamás un `decide()`.

| Propiedad | Tipo | Qué previene | Ejemplo de violación |
| :--- | :--- | :--- | :--- |
| **Agreement** | Safety | Divergencia e inconsistencia | Dos réplicas deciden valores distintos (Split-Brain) |
| **Validity** | Safety | Decisiones espurias o triviales | El sistema decide un valor que nadie propuso |
| **Termination** | Liveness | Bloqueo indefinido (Deadlock / Livelock) | Los nodos compiten eternamente y nunca deciden |

---

3) **Por qué el consenso es necesario en sistemas replicados (Relación con la Clase 03)**

En la clase anterior vimos que para tolerar fallas y mejorar la disponibilidad y latencia, los sistemas recurren a la replicación de datos. Sin embargo, nos encontramos con un problema central: **el estado**, y la dificultad fundamental de que **no se pueden mantener todas las copias sincronizadas sin coordinación**.

En la Clase 03 observamos:
1. **Escrituras concurrentes y divergencia:** En arquitecturas Multi-Leader o Leaderless, dos clientes pueden escribir concurrentemente sobre réplicas distintas ($x=1$ en réplica A y $x=2$ en réplica B). Para resolver el conflicto se recurre a parches como *Last Write Wins* (LWW), que descarta datos arbitrariamente debido al desfase de relojes físicos, o a *Vector Clocks*, que solo detectan la concurrencia pero no la resuelven de forma automática y transparente.
2. **Elección de líder y Split-Brain:** En sistemas basados en líder (Leader-based), si el líder falla, es obligatorio elegir uno nuevo. Pero si no hay consenso para determinar quién es el nuevo líder, una partición de red puede provocar que dos nodos se crean líderes al mismo tiempo, aceptando escrituras contradictorias.
3. **Máquina de Estados Replicada (State Machine Replication - SMR):** Para lograr la consistencia más fuerte (**Linearizabilidad**), todas las réplicas deben ejecutar exactamente la misma secuencia ordenada de operaciones:

$$\text{Log}[1] \to \text{Log}[2] \to \dots \to \text{Log}[k]$$

Ponerse de acuerdo sobre cuál comando exacto ocupa la posición $k$ del log es, precisamente, una instancia del **problema de consenso**. El consenso es el pegamento que permite que múltiples réplicas físicas se comporten lógicamente como una única máquina confiable.

---

4) **Ejecución con 3 nodos donde dos deciden valores distintos**

* **Supuestos:** Sistema con 3 nodos ($N_1, N_2, N_3$). Protocolo ingenuo que decide sin verificar una mayoría estricta global o ante una partición de red no detectada.
* **Modelo de fallas:** Partición de red que divide el cluster en dos componentes aislados: $\{N_1\}$ por un lado y $\{N_2, N_3\}$ por el otro.

**Construcción de la ejecución:**
1. Estado inicial: $x = 0$.
2. El cliente $C_A$ le envía a $N_1$ la propuesta `v = A`.
3. El cliente $C_B$ le envía a $N_2$ la propuesta `v = B`.
4. Ocurre una partición de red: $N_1$ queda completamente aislado de $N_2$ y $N_3$.
5. $N_1$, aplicando una regla ingenua local (o timeout sin requerir quórum mayoritario de confirmación), decide de forma aislada: $\text{decide}_1(A)$.
6. $N_2$ y $N_3$ se comunican entre sí, alcanzan un quórum local y deciden: $\text{decide}_2(B)$ y $\text{decide}_3(B)$.

```
Tiempo ─────────────────────────────────────────────────────────────►

N1 (aislado):      Recibe prop(A) ───► [ decide(A) ]
                      X (partición de red)
N2:                Recibe prop(B) ───► Consulta N3 ──► [ decide(B) ]
N3:                                      Responde OK  ──► [ decide(B) ]
```

* **Evidencia observable:** El nodo $N_1$ responde a $C_A$ confirmando `A`, mientras que $N_2$ responde a $C_B$ confirmando `B`.
* **Propiedad violada:** Se viola **Agreement (Acuerdo)**, dado que existen dos nodos no fallidos que emitieron decisiones distintas ($N_1$ decidió $A$ y $N_2$ decidió $B$). El sistema sufre una inconsistencia crítica de Split-Brain.

---

5) **Definición de estado univalente y bivalente**

En el análisis formal de algoritmos de consenso (marco teórico de Fischer, Lynch y Paterson - FLP):

Una **configuración o estado global $C$** del sistema está compuesto por los estados internos de todos los procesos más el conjunto de mensajes que se encuentran en tránsito en la red.

* **Estado Univalente:** Una configuración $C$ es *univalente* si todas las ejecuciones posibles que parten desde $C$ conducen inevitablemente a decidir el **mismo valor**. La decisión final del sistema ya está completamente determinada, sin importar el orden futuro de entrega de mensajes o las velocidades relativas de los nodos.
  * **0-valente:** Cualquier ejecución válida futura sólo puede culminar con la decisión del valor `0`.
  * **1-valente:** Cualquier ejecución válida futura sólo puede culminar con la decisión del valor `1`.

* **Estado Bivalente:** Una configuración $C$ es *bivalente* si el resultado final aún está indefinido. Es decir, a partir de $C$, existe al menos una secuencia de pasos que lleva al sistema a decidir `0`, y **también** existe otra secuencia de pasos válida que lleva al sistema a decidir `1`.

```
                    ┌─── [Pasos E1] ───► Configuración 0-valente ───► decide(0)
Configuración       │
Bivalente (C) ──────┤
                    └─── [Pasos E2] ───► Configuración 1-valente ───► decide(1)
```

> **En criollo:** Un estado bivalente es una moneda en el aire: todavía puede caer cara o cruz según cómo sople el viento (el scheduler). Un estado univalente es cuando la moneda ya cayó contra la mesa: aunque un nodo todavía no haya mirado el piso, el resultado ya es irreversible.

---

## Nivel 2: ejecuciones y fallas

1) **Por qué el estado inicial puede ser bivalente**

Considérese un sistema con entradas binarias $\{0, 1\}$. La configuración inicial del sistema depende del vector de propuestas de cada proceso: $C_{init} = (v_1, v_2, \dots, v_n)$.

1. Si todos los procesos comienzan proponiendo `0`, es decir $C_{(0)} = (0, 0, \dots, 0)$, por la propiedad de **Validity**, la única decisión legítima posible es `0`. Por lo tanto, $C_{(0)}$ es forzosamente **0-valente**.
2. Si todos los procesos comienzan proponiendo `1`, es decir $C_{(1)} = (1, 1, \dots, 1)$, por **Validity**, la única decisión legítima posible es `1`. Por lo tanto, $C_{(1)}$ es forzosamente **1-valente**.
3. Si cambiamos las entradas de los procesos una por una desde $C_{(0)}$ hasta $C_{(1)}$, se pasa de una configuración 0-valente a una 1-valente. En este trayecto debe existir una configuración con entradas mixtas (por ejemplo, $p_1$ tiene entrada `0` y $p_2$ tiene entrada `1`).
4. Consideremos una configuración con entradas mixtas:
   * Si el proceso que tiene entrada `0` se cae inmediatamente al inicio sin enviar ningún mensaje, los procesos restantes deben ser capaces de avanzar y terminar (por la propiedad de **Termination**), y como solo conocen propuestas `1`, deben decidir `1`.
   * Por el contrario, si es el proceso con entrada `1` el que se cae al inicio, los nodos supervivientes solo conocen propuestas `0` y deben decidir `0`.
5. Como a partir de la misma configuración inicial existen historias perfectamente válidas donde el sistema decide `0` e historias donde decide `1`, se concluye que **el estado inicial con entradas mixtas es intrínsecamente bivalente**.

---

2) **El rol del scheduler adversario en la prueba FLP**

En el modelo asíncrono, no existen nociones de tiempo absoluto ni cotas máximas para el retardo de la red. El orden y el momento exacto en que los mensajes en tránsito son entregados a los procesos queda determinado por una entidad abstracta denominada **scheduler** (planificador).

En la demostración del teorema de imposibilidad de **FLP (Fischer, Lynch, Paterson, 1985)**:
* El **scheduler adversario** es una construcción matemática que actúa como un "oponente" del algoritmo de consenso.
* Posee **visión omnisciente**: conoce el estado interno de cada nodo y todos los mensajes que viajan por la red.
* **Su objetivo:** Demostrar que ningún algoritmo determinista puede garantizar consenso perfecto eligiendo estratégicamente qué mensaje entregar y cuándo, para evitar que el sistema decida.
* **Cómo opera:** Sabe que para decidir, el sistema debe transicionar obligatoriamente de un estado *bivalente* a uno *univalente*. Justo cuando la entrega de un mensaje $m$ está a punto de forzar al sistema a volverse univalente, el scheduler adversario intercala la entrega de otro mensaje o demora $m$ de forma tal que el sistema aterrice en otra configuración que sigue siendo **bivalente**.
* Al repetir esta maniobra indefinidamente, el adversario mantiene al sistema atrapado en una oscilación eterna entre estados bivalentes sin violar ninguna regla del modelo (todos los mensajes son eventualmente entregados, pero en el orden más desfavorable posible).

---

3) **Construcción de una ejecución donde el sistema nunca decide (Livelock infinito)**

* **Supuestos:** Red asíncrona no particionada, 2 nodos ($p_1, p_2$), modelo de fallas crash-stop (ningún nodo cae realmente, pero la red es arbitrariamente lenta).
* **Configuración inicial:** Bivalente ($p_1$ propone `0`, $p_2$ propone `1`).

**Trazabilidad paso a paso:**
1. **Ronda 1:** $p_1$ intenta fijar el valor `0` y envía el mensaje $m_1$ a $p_2$.
2. El scheduler adversario retiene $m_1$ en el canal. Como la red es asíncrona, para $p_2$ es imposible distinguir si $p_1$ murió o si el paquete viene lento.
3. Para no violar *Termination*, $p_2$ no puede esperar eternamente: cambia a modo proponente e intenta fijar su propio valor `1`, enviando $m_2$ hacia $p_1$.
4. El adversario libera $m_1$ justo antes de que $p_2$ confirme su decisión. $p_2$ recibe $m_1$, detecta el conflicto y aborta su intento de decidir `1` para preservar *Agreement*, retrocediendo a un estado de duda.
5. El adversario ahora retiene $m_2$ hacia $p_1$. $p_1$ se impacienta, supone que $p_2$ no respondió y genera una nueva propuesta con número de ronda superior ($m_3$).
6. Justo en ese instante, el adversario le entrega $m_2$ a $p_1$, desestabilizando a $p_1$ y forzándolo a cancelar su avance.
7. El sistema ha completado un ciclo cerrado y regresa a una configuración **bivalente**.

```
p1 (propone 0)                                   p2 (propone 1)
     │                                                │
     ├─── m1 ("voto por 0") ────────────────────────X │ (retenido por adversario)
     │                                                │
     │                                                ├─── m2 ("voto por 1") ──┐
     │                                                │                        │ (retenido)
     │                               ◄── m1 llega ────┤ (p2 cancela y resetea) │
     │                                                │                        │
     ├─── m3 ("ronda 2 por 0") ─────────────────────X │                        │
     │                                                │                        │
     │◄── m2 llega ───────────────────────────────────┼────────────────────────┘
  (p1 cancela y resetea)                              │
     │                                                │
     ▼                                                ▼
     [ Estado sigue BIVALENTE ] ──► Se repite el ciclo ad infinitum
```

* **Evidencia observable:** El tráfico de red crece continuamente (se intercambian infinitos mensajes), los contadores internos de rondas aumentan monótonamente, pero ningún nodo emite jamás la llamada al evento `decide()`. Se viola **Termination**.

---

4) **Por qué agregar timeouts rompe el modelo de FLP**

El teorema de FLP se sustenta de manera intransigente sobre el supuesto de **asincronía pura**:
* En un entorno puramente asíncrono, **el tiempo no existe como fuente de información**. No hay límites conocidos para la demora de los mensajes ni para la velocidad de procesamiento.
* Consecuencia teórica: si un nodo $p_i$ tarda 100 horas en responder, es **matemáticamente imposible** para los demás nodos distinguir si $p_i$ sufrió un crash definitivo o si simplemente la red está congestionada.

**¿Por qué los timeouts rompen este esquema?**
1. **Cambio de modelo físico (Sincronía Parcial):** Al introducir un temporizador (timeout), se abandona la asincronía pura y se asume un modelo de **sincronía parcial** (definido por Dwork, Lynch y Stockmeyer) o la presencia de un **detector de fallas no confiable** (Chandra y Toueg).
2. **Cota de tiempo ($\Delta$):** Se asume que existe un tiempo de estabilización global (Global Stabilization Time - GST) tras el cual los mensajes tardan a lo sumo un tiempo acotado $\Delta$.
3. **Pérdida de poder del adversario:** Si un nodo no responde dentro de una ventana de timeout calculada, los nodos restantes están facultados para **sospechar de su caída**, descartarlo y avanzar en una nueva ronda sin temer que el mensaje demorado destruya la validez del estado.
4. **Cómo lo aprovechan los protocolos prácticos (Raft / Paxos):**
   * Mantienen **Safety (Agreement)** de forma incondicional incluso bajo asincronía total (nunca deciden dos valores distintos aunque los timeouts fallen estrepitosamente).
   * Sacrifican **Liveness temporalmente** cuando la red es errática, pero garantizan **Termination** tan pronto como la red vuelve a comportarse dentro de los límites del timeout.

---

5) **Proceso detallado de elección de líder en Raft**

En Raft, los nodos se encuentran en uno de tres roles: **Follower**, **Candidate** o **Leader**.

```
    ┌──────────────────────────────────────────────┐
    │                                              │ (descubre líder con term mayor)
    ▼                                              │
[ Follower ] ──(election timeout)──► [ Candidate ] ──(gana mayoría)──► [ Leader ]
    ▲                                      │                               │
    │                                      │ (otro nodo gana la elección) │
    └──────────────────────────────────────┴───────────────────────────────┘
```

**Paso a paso del proceso de elección:**

1. **Monitoreo de Heartbeats:**
   * En estado normal, los followers reciben periódicamente señales de vida (heartbeats vacíos vía RPC `AppendEntries`) del líder actual.
   * Cada follower posee un temporizador interno llamado **election timeout**.

2. **Disparo de la elección (Transición a Candidate):**
   * Si un follower deja de recibir heartbeats durante su election timeout, asume que el líder cayó o quedó incomunicado.
   * El follower cambia su rol a **Candidate**:
     1. Incrementa su término actual: `currentTerm = currentTerm + 1`.
     2. Vota por sí mismo: `votedFor = self.id`.
     3. Reinicia su election timer.
     4. Envía en paralelo el RPC `RequestVote(term, candidateId, lastLogIndex, lastLogTerm)` a todos los demás nodos del cluster.

3. **Criterios de votación en los receptores:**
   Al recibir un `RequestVote`, cada nodo evalúa estrictamente dos condiciones para conceder su voto:
   * **Regla del Término y Voto Único:** El término recibido debe ser $\ge \text{currentTerm}$ local, y el nodo no debe haber votado por ningún otro candidato en ese término (`votedFor == null` o `votedFor == candidateId`).
   * **Regla de Actualización del Log (Up-to-Date check):** Raft exige que el candidato posea un log al menos tan completo como el del votante para no perder entradas commiteadas. Se comparan los últimos registros:
     * Si `lastLogTerm` del candidato es mayor al local $\implies$ voto concedido.
     * Si los `lastLogTerm` son iguales, pero `lastLogIndex` del candidato es $\ge$ al local $\implies$ voto concedido.
     * En caso contrario $\implies$ voto rechazado.

4. **Desenlaces posibles de la elección:**
   * **Gana la elección:** El candidato acumula votos de una **mayoría estricta** de nodos ($\lfloor n/2 \rfloor + 1$). Inmediatamente se proclama **Leader** y envía heartbeats a todo el cluster para sofocar otras candidaturas.
   * **Reconoce a otro líder:** Mientras espera votos, recibe un `AppendEntries` de otro nodo con un término $\ge \text{currentTerm}$. Reconoce al nuevo líder legítimo y regresa a ser **Follower**.
   * **Split Vote (Empate / Votos divididos):** Si múltiples seguidores se postulan al mismo tiempo y ninguno logra alcanzar la mayoría (por ejemplo, en un cluster de 4 nodos se reparten 2 y 2), la elección fracasa por timeout.

> **Mecanismo de defensa contra Split Votes:** Raft asigna timeouts de elección **aleatorizados** (típicamente entre 150 ms y 300 ms) en cada nodo. Esto desincroniza a los nodos, asegurando que casi siempre un único nodo despierte primero, inicie la votación y junte la mayoría antes de que los demás expiren.

---

## Nivel 3: diseño y comparación

1) **El rol de los términos (terms)**

En Raft, los **términos** cumplen la función de **relojes lógicos** (equivalentes a los timestamps lógicos de Lamport):
* El tiempo se subdivide en términos arbitrarios representados por enteros monótonamente crecientes ($1, 2, 3, \dots$).
* Cada término comienza con un período de elección. Si la elección es exitosa, ese término queda bajo el mando de un único líder hasta que termine su mandato.

**Funciones críticas que cumplen los términos:**
1. **Detección y neutralización de líderes obsoletos (Fencing):** Si un líder viejo queda aislado por una partición de red, el resto del cluster avanza al término $t+1$ y elige un nuevo líder. Cuando la red se restablece y el líder viejo intenta mandar mensajes con término $t$, los receptores rechazan sus RPCs porque $t < \text{currentTerm}$. Al recibir el rechazo, el líder viejo descubre que su término caducó y se degrada inmediatamente a Follower.
2. **Unicidad de liderazgo:** Las reglas de Raft garantizan que en un término determinado puede existir **a lo sumo un único líder electo**, evitando bifurcaciones concurrentes en una misma época.
3. **Sincronización del estado del log:** Cada entrada guardada en el log lleva impreso el término en el que fue propuesta. Esto le permite al protocolo validar la consistencia del historial entre diferentes réplicas.

---

2) **Descripción del protocolo AppendEntries**

El RPC `AppendEntries` es el mecanismo central de Raft. Es invocado exclusivamente por el **Líder** y cumple una doble función: replicar entradas de log de las transacciones de los clientes y funcionar como **heartbeat** periódico (cuando la lista de entradas está vacía).

**Parámetros enviados por el Líder:**
* `term`: Término actual del líder.
* `leaderId`: Identificador del líder (para que los seguidores redirijan a los clientes).
* `prevLogIndex`: Índice del registro inmediatamente anterior a las nuevas entradas que se van a anexar.
* `prevLogTerm`: Término del registro ubicado en `prevLogIndex`.
* `entries[]`: Vector de entradas de log a replicar (vacío para heartbeats).
* `leaderCommit`: Índice de la última entrada confirmada (commiteada) por el líder.

**Respuesta devuelta por el Seguidor:**
* `term`: Término actual del seguidor (para que el líder se actualice si quedó atrasado).
* `success`: Booleano (`true` si el seguidor contenía una entrada coincidente en `prevLogIndex` con término `prevLogTerm`).

**Lógica de validación en el Follower (Paso a paso):**
1. Si `term < currentTerm` local $\implies$ devuelve `success = false`.
2. **Chequeo de consistencia inductiva:** Si el seguidor no tiene ninguna entrada en `prevLogIndex` cuyo término coincida exactamente con `prevLogTerm` $\implies$ devuelve `success = false`.
3. **Resolución de conflictos:** Si una entrada existente en el log choca con una nueva (mismo índice pero distinto término), el seguidor elimina esa entrada conflictiva y todas las subsiguientes.
4. **Anexión:** Agrega cualquier entrada nueva de `entries[]` que aún no estuviera en su log.
5. **Avance del commit:** Si `leaderCommit > commitIndex`, actualiza su commit local al valor mínimo: `commitIndex = min(leaderCommit, índice del último registro nuevo)`.

```
LÍDER (Term 2)                                 FOLLOWER
      │                                            │
      ├─── AppendEntries(prevIndex=4, ────────────►│ ¿Log[4].term == 2?
      │                  prevTerm=2,               │ SÍ: anexa nuevas entradas,
      │                  entries=[...])            │     devuelve success = true.
      │◄── Response(success=true) ────────────────┤ NO: rechaza, devuelve false.
```

Si el seguidor devuelve `success = false` por fallo de consistencia, el líder decrementa progresivamente el puntero `nextIndex` correspondiente a ese seguidor y reintenta enviar el RPC hasta encontrar el punto exacto donde los logs coinciden, sobreescribiendo a partir de allí cualquier divergencia histórica.

---

3) **Definición formal de la propiedad de Log Matching**

La propiedad de **Log Matching** garantiza la coherencia total de los logs replicados a lo largo del tiempo. Formalmente, establece dos condiciones:

1. **Cláusula 1 (Identidad local):** Si dos entradas en logs de diferentes nodos tienen el **mismo índice** y el **mismo término**, entonces ambas entradas almacenan **exactamente el mismo comando** de la máquina de estados.
2. **Cláusula 2 (Identidad de prefijos):** Si dos entradas en logs de diferentes nodos tienen el **mismo índice** y el **mismo término**, entonces los logs de ambas réplicas son **estrictamente idénticos en todas las entradas precedentes** (desde el índice $1$ hasta el índice $i-1$).

$$\forall A, B \quad (\text{Log}_A[i].\text{term} = \text{Log}_B[i].\text{term}) \implies (\text{Log}_A[i].\text{cmd} = \text{Log}_B[i].\text{cmd} \;\land\; \forall k < i, \text{Log}_A[k] = \text{Log}_B[k])$$

* **Cómo se sostiene formalmente:**
  * La Cláusula 1 se cumple porque un líder solo crea una única entrada por índice en un término determinado, y el líder nunca modifica ni borra sus propias entradas.
  * La Cláusula 2 se garantiza por **inducción** a través del chequeo de consistencia de `AppendEntries`: un seguidor nunca acepta una entrada en el índice $i$ a menos que verifique exitosamente que su log en $i-1$ coincide exactamente con el del líder.

---

4) **Por qué el commit requiere mayoría**

En Raft, una entrada se considera **commiteada** (comprometida / confirmada de forma segura) una vez que el líder que la propuso ha logrado replicarla con éxito en una **mayoría absoluta** de los nodos del cluster ($\lfloor n/2 \rfloor + 1$).

**Fundamento matemático (Intersección de Quórums):**
En cualquier conjunto de $n$ nodos, cualesquiera dos subconjuntos mayoritarios $Q_1$ y $Q_2$ deben intersectarse necesariamente en al menos un nodo en común:

$$Q_1 \cap Q_2 \neq \emptyset$$

```
   ┌───────────────────────┐
   │ Quórum de Commit (Q1) │
   │      { N1, N2 }       │
   └───────────┬───────────┘
               │ ◄─── Nodo en común (N2): conoce la entrada commiteada
   ┌───────────┴───────────┐
   │ Quórum de Elec. (Q2)  │
   │      { N2, N3 }       │
   └───────────────────────┘
```

**Por qué esto preserva la consistencia:**
1. Supóngase que una entrada se commitea en una mayoría $Q_1$.
2. Si el líder actual cae, para que cualquier nuevo nodo sea electo líder en un término superior, debe reunir los votos de otra mayoría $Q_2$.
3. Por el principio de casillas o palomar, **existe al menos un nodo $x$ que pertenece simultáneamente a $Q_1$ y a $Q_2$**.
4. Como $x \in Q_1$, $x$ almacena obligatoriamente en su disco la entrada commiteada.
5. Gracias a la regla de votación de Raft (Up-to-Date check), el nodo $x$ **solo votará** por un candidato cuyo log esté al menos tan actualizado como el suyo.
6. En consecuencia, es imposible que gane la elección un candidato que no contenga la entrada commiteada (**Leader Completeness Property**).

* **Trade-off:** Exigir mayoría asegura consistencia ante caídas de hasta $f = \lfloor (n-1)/2 \rfloor$ nodos. El costo es que si cae una mayoría estricta de nodos, el sistema deja de aceptar nuevas escrituras para preservar la seguridad (prioriza Consistencia sobre Disponibilidad según el Teorema CAP).

---

5) **Construcción de una ejecución donde un líder cae y otro lo reemplaza**

* **Supuestos:** Cluster de 3 nodos ($S_1, S_2, S_3$). Inicialmente en Término 1 con $S_1$ como Líder. Todos los logs tienen una entrada previa en índice 1 (`[1: "x=5"]`).
* **Modelo de fallas:** Falla por caída abrupta (crash-stop) del líder $S_1$.

**Traza de la ejecución paso a paso:**

```
Tiempo ─────────────────────────────────────────────────────────────────────────────►

S1 (Líder T1): ──[ CRASH ]
                    X
S2 (Follower):   Heartbeat timeout expira ──► Pasa a CANDIDATE (Term 2)
                 Vota por sí mismo
                 Envía RequestVote(term=2, lastLogIdx=1, lastLogTerm=1) ──┐
                                                                          │
S3 (Follower):   Recibe RequestVote de S2 ◄───────────────────────────────┘
                 Verifica: Term 2 > 1, Log actualizado ──► Otorga Voto (true)
                 S2 acumula 2 votos de 3 (Mayoría alcanzada)
                 S2 se proclama LÍDER de Término 2
                 S2 envía Heartbeat AppendEntries(term=2, leaderId=S2) ──► S3 resetea timer
```

1. **Estado Inicial:** $S_1$ es líder en término 1. Envía heartbeats periódicos a $S_2$ y $S_3$.
2. **Falla:** $S_1$ sufre un fallo de hardware y se apaga por completo.
3. **Detección:** $S_2$ y $S_3$ dejan de recibir heartbeats. Sus election timers avanzan.
   * El timer de $S_2$ fue aleatorizado a 170 ms.
   * El timer de $S_3$ fue aleatorizado a 260 ms.
4. **Inicio de elección:** A los 170 ms expira el temporizador de $S_2$. $S_2$ transiciona a Candidate:
   * Setea `currentTerm = 2`.
   * Vota por sí mismo (`votedFor = S2`).
   * Envía `RequestVote(term=2, candidateId=S2, lastLogIndex=1, lastLogTerm=1)` a $S_3$ (y a $S_1$, que no responde).
5. **Evaluación de voto en $S_3$:** A los 180 ms, $S_3$ recibe la solicitud de $S_2$:
   * Observa `term = 2 > currentTerm(1)`, por lo que actualiza su término local a 2.
   * Chequea su estado: aún no votó en el término 2.
   * Compara logs: el log de $S_2$ (índice 1, término 1) está tan actualizado como el suyo.
   * $S_3$ otorga su voto a $S_2$ y resetea su election timer.
6. **Consagración del nuevo líder:** $S_2$ contabiliza 2 votos ($S_2$ y $S_3$) sobre un total de 3 nodos. Como reúne mayoría absoluta, asume inmediatamente como **Líder del Término 2**.
7. **Estabilización observable:** $S_2$ despacha de inmediato un `AppendEntries(term=2, leaderId=S2)` vacío a $S_3$. $S_3$ procesa el mensaje, confirma a $S_2$ como su nuevo líder y el cluster vuelve a operar con normalidad.

---

## Nivel 4: integración y defensa

1) **Los roles de Paxos: Proposer, Acceptor, Learner**

En el protocolo Paxos básico (diseñado por Leslie Lamport), el consenso se resuelve mediante la interacción coordinada entre tres roles funcionales:

* **Proposer (Proponente):**
  * Actúa en representación de los clientes que desean ejecutar una operación.
  * Su función es impulsar un valor intentando que los acceptors lo adopten.
  * Genera un identificador de propuesta único y monótono $n$, y lidera las dos fases del protocolo (`Prepare` y `Accept`).
* **Acceptor (Aceptador):**
  * Constituye la **memoria y autoridad del consenso**.
  * Los acceptors se agrupan formando quórums mayoritarios.
  * Almacenan de forma duradera las promesas realizadas y los valores que han aceptado a lo largo de las rondas.
  * Votan si aceptan o rechazan las propuestas de los proposers para proteger la seguridad del acuerdo.
* **Learner (Aprendiz):**
  * No interviene activamente en la votación.
  * Su tarea es detectar cuándo un valor ha sido aceptado por una mayoría de acceptors (**valor elegido**) para luego ejecutarlo en la máquina de estados local y notificar el resultado final al cliente.

| Rol | Función Principal | Estado persistente clave |
| :--- | :--- | :--- |
| **Proposer** | Conduce la ronda y propone valores | Número de propuesta asignado ($n$) |
| **Acceptor** | Vota y resguarda la validez del quórum | $\text{minProposal}$, $(n_{accepted}, v_{accepted})$ |
| **Learner** | Ejecuta la decisión acordada | Registro de acuerdos confirmados |

*(En implementaciones industriales como Chubby o Spanner, cada servidor físico ejecuta simultáneamente los tres roles).*

---

2) **La fase Prepare (Fase 1 de Paxos)**

El objetivo de la Fase 1 es doble: adquirir el derecho a proponer garantizando exclusividad sobre números de propuesta menores, y descubrir si el sistema ya aceptó algún valor en rondas previas.

**Paso 1a: Solicitud `Prepare(n)` del Proposer**
* El proposer escoge un número de propuesta $n$ que debe ser estrictamente mayor a cualquier número que haya utilizado antes, y único en el cluster (por ejemplo, combinando un contador con su Node ID: $n = \langle \text{contador}, \text{nodeId} \rangle$).
* Envía el mensaje `Prepare(n)` a una mayoría de acceptors.

**Paso 1b: Respuesta `Promise` del Acceptor**
Cada acceptor que recibe `Prepare(n)` evalúa $n$ contra su variable local $\text{minProposal}$ (la propuesta más alta que haya visto hasta el momento):
* **Si $n > \text{minProposal}$:**
  1. Actualiza $\text{minProposal} = n$.
  2. Emite una **promesa formal**: promete no aceptar jamás ninguna propuesta futura que tenga un número menor a $n$.
  3. Devuelve en su respuesta el par $(n_{max}, v_{max})$ correspondiente a la propuesta de mayor número que ya haya aceptado en el pasado (si nunca aceptó ninguna, devuelve `null`).
* **Si $n \le \text{minProposal}$:** El acceptor ignora el mensaje o responde con un `Reject/NACK`, indicando que la propuesta es obsoleta.

```
PROPOSER                                       ACCEPTOR
   │                                              │
   ├─── Prepare(n) ──────────────────────────────►│ ¿n > minProposal?
   │                                              │ SÍ: minProposal = n
   │                                              │     devuelve Promise(n, n_max, v_max)
   │◄── Promise(n, n_max, v_max) ─────────────────┤ NO: rechaza / ignora
```

---

3) **La fase Accept (Fase 2 de Paxos)**

Una vez que el proposer obtuvo promesas de una mayoría de acceptors, procede a intentar consolidar el valor.

**Paso 2a: Petición `Accept(n, v)` del Proposer**
* Antes de enviar el valor, el proposer debe seleccionarlo acatando una regla de seguridad inquebrantable:
  * Examina todas las respuestas `Promise` recibidas de la mayoría:
    * Si al menos un acceptor devolvió un valor previamente aceptado $(n_{max}, v_{max})$, el proposer **está obligado por el protocolo a adoptar como $v$ el valor asociado al $n_{max}$ más alto reportado**.
    * Si ningún acceptor reportó haber aceptado valores antes (todas las respuestas tenían `null`), el proposer es libre de proponer el valor original solicitado por su cliente.
* El proposer envía el mensaje `Accept(n, v)` a la mayoría de acceptors.

**Paso 2b: Aceptación `Accepted(n, v)` del Acceptor**
Cuando un acceptor recibe `Accept(n, v)`:
* Evalúa si ha recibido con posterioridad un `Prepare(n')` con $n' > n$:
  * **Si $n \ge \text{minProposal}$:** El acceptor acepta la propuesta. Registra en disco persistente $n_{accepted} = n$ y $v_{accepted} = v$, y envía un mensaje `Accepted(n, v)` tanto al proposer como a los learners.
  * **Si $n < \text{minProposal}$:** Significa que otro proposer inició una ronda con número superior y este acceptor ya prometió rechazar números viejos. Por ende, rechaza la propuesta.

---

4) **Definición formal de cuándo un valor es elegido (chosen)**

En Paxos, la definición formal de elección establece:

> Un valor $v \in V$ se considera **elegido (chosen)** si y solo si ha sido aceptado (en la Fase 2b) por un **quórum mayoritario de acceptors** en alguna ronda o propuesta $n$.

$$\text{Chosen}(v) \iff \exists n, \exists Q \subseteq \text{Acceptors} \quad \text{tal que} \quad |Q| > \frac{|\text{Acceptors}|}{2} \quad \land \quad \forall a \in Q, \ a \ \text{aceptó} \ (n, v)$$

**Implicancia fundamental:**
* La condición de "elegido" es una propiedad intrínseca del estado del sistema, no del conocimiento de los nodos.
* En el instante milimétrico en que el último acceptor de la mayoría registra $(n, v)$ en su almacenamiento, el valor $v$ queda **sellado e inmutable para toda la eternidad**. Aunque el proposer se caiga en ese mismo microsegundo sin enterarse, ningún otro valor $v' \neq v$ podrá jamás ser elegido por el cluster.

---

5) **Por qué dos valores distintos no pueden ser elegidos**

Esta es la demostración central de la propiedad de **Agreement** en Paxos:

**Teorema:** Si un valor $v$ es elegido en la propuesta $n$, entonces cualquier propuesta con número $n' > n$ que sea emitida por cualquier proponente contendrá necesariamente el mismo valor $v$.

**Demostración por inducción y contradicción (Quorum Intersection):**
1. Supóngase por el absurdo que se eligen dos valores diferentes $v_1 \neq v_2$ en dos propuestas con números $n_1 < n_2$, donde $n_1$ fue la primera propuesta en elegir un valor.
2. Como $v_1$ fue elegido en $n_1$, existe un quórum mayoritario de acceptors $Q_1$ tal que todos los nodos en $Q_1$ aceptaron $(n_1, v_1)$.
3. Para que el proposer de la ronda $n_2$ haya podido enviar un mensaje `Accept(n_2, v_2)`, tuvo que haber completado previamente con éxito la Fase 1 (Prepare), reuniendo las respuestas `Promise` de otro quórum mayoritario de acceptors $Q_2$.
4. **Propiedad de intersección de quórums:** Como $Q_1$ y $Q_2$ son dos mayorías de un mismo conjunto de acceptors, su intersección nunca puede ser vacía:

$$Q_1 \cap Q_2 \neq \emptyset$$

5. Sea $a \in Q_1 \cap Q_2$ un acceptor perteneciente a ambos quórums:
   * Dado que $a \in Q_1$, el nodo $a$ aceptó la propuesta $(n_1, v_1)$.
   * Dado que $a \in Q_2$, el nodo $a$ respondió con una promesa `Promise` al proposer de $n_2$. Como $n_2 > n_1$, cuando $a$ envió su promesa, ya había aceptado previamente $(n_1, v_1)$.
6. Por la regla de la Fase 1b, el nodo $a$ incluyó obligatoriamente en su respuesta el par $(n_1, v_1)$ que había aceptado.
7. Por la regla de la Fase 2a, el proposer de $n_2$ está forzado por construcción a inspeccionar todas las respuestas y **adoptar el valor asociado al mayor número de propuesta reportado**.
8. Como el valor de mayor número reportado es $v_1$ (o una propuesta intermedia que por inducción ya portaba $v_1$), el proposer de $n_2$ **no tiene permitido proponer su propio valor ni ningún valor diferente de $v_1$**. Está forzado a enviar `Accept(n_2, v_1)`.
9. En consecuencia, es matemáticamente imposible que un valor $v_2 \neq v_1$ sea propuesto o aceptado por una mayoría en la ronda $n_2$.

**Conclusión:** Dos valores distintos nunca pueden ser elegidos. La seguridad (Safety / Agreement) de Paxos se mantiene estrictamente garantizada bajo cualquier combinación de retardos de red, particiones o caídas de nodos.
