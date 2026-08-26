## Nivel 1: fundamentos y reconocimiento

https://www.geeksforgeeks.org/operating-systems/what-is-replication-in-distributed-system/

1) La replicación en sistemas distribuídos se pueede definir como el proceso de crear y mantener múltiples copias de datos, recursos o servicios en diferentes 
nodos (sean estos nodos o servidores) dentro de una red. El objetivo es mejorar la confiabilidad, la disponibilidad y rendimietno del sistema, aseguranod que los 
datos o servicios sean accesibles incluso si algunos nodos fallan o dejan de estar disponibles.

Enceuntra su origen en un problema central: el estado. Para mejorar disponibilidad y latencia, los sistemas replican datos. Introduce una dificultad fundamental:

**no se pueden mantener todas las copias perfectametne sinronizadas sin coordinación**

2) La divergencia es inevitable debido a la latencia de la red y la posiblidad de particoines de red, lo que impide que múltiples nodos mantengan un estado ide´ntico de manera instantánea y simultánea.

3) 
| Momento               | Réplica A | Réplica B |
| --------------------- | --------- | --------- |
| Inicial               | `x = 0`   | `x = 0`   |
| Escritura concurrente | `x = 1`   | `x = 2`   |
| Resultado             | `x = 1`   | `x = 2`   |

4) Estado físico: lo que realmente está almacenado en cada réplica en ese momento
Puede diferir entre las réplicas

Estado lógico: estado que el sistema considera que represetna correctamente los datos, independeintementee de cómo estén distribuídos físicamente
Si el sistema detecta una escritura concurrente entre x=1 y x=2 pude considerar:

```Estado lógico --> x tiene un conflicto entre 1 y 2 ```

y posteriormente resolverlo, mediante Last Write Wins como:

```estado lógico final --> x=2```

Aunque temporalmetne las réplcias aún estén defasadas. 

|                              | Estado físico                 | Estado lógico                                    |
| ---------------------------- | ----------------------------- | ------------------------------------------------ |
| Qué representa               | Lo almacenado en cada réplica | El estado que el sistema considera válido        |
| Puede diferir entre réplicas | **Sí**                        | Idealmente **no**, una vez resuelto el conflicto |
| Ejemplo                      | A: `x=1`, B: `x=2`            | `x=2` después de resolver                        |
| Depende de                   | Estado local de cada réplica  | Reglas de consistencia/conflict resolution       |

El estado físico corresponde a los datos actualmente almacenados en cada réplica, mientras que el estado lógico representa la visión consistente del sistema sobre esos datos, independientemente de las diferencias temporales entre las réplicas.
⛔ Fïsicamente no está en ningún lugar el lógico, cierto?

5) Porque ante cualquier duda (ya que ningún nodo está exento de sufrir degradación de la información)
si sólo tengo una sola fuente de verdad y se corrompe no puedo reconstruir la información correcta o "desempatar". Mínimo y de preferencia 3 nodos par decidir cual tiene razón

Una sola fuente de verdad es difícil de sostener ante una partición porque los demás nodos pueden quedar incomunicados con ella. Si el nodo que actúa como fuente de verdad queda aislado, los otros nodos no pueden consultar ni actualizar ese estado, lo que afecta la disponibilidad. Si, en cambio, se permite que ambos lados de la partición acepten escrituras, pueden generarse estados divergentes que luego deben reconciliarse. Por eso los sistemas replicados utilizan múltiples réplicas y mecanismos de consenso o resolución de conflictos.

## Nivel 2: ejecuciones y fallas

1) Linearizabilidad: garantía de consistencia fuerte en sistemas concurrentes y distribuídos, donde cada operación parece ocurrir de forma instantánea en un punto específico del tiempo real entre su invocación y su respuesta
Ejemplo (registro compartido): x = 0

Cliente A: WRITE(x, 1)
Cliente B: READ(x)

Tiempo ──────────────────────────────>

A:  WRITE(x,1) ──────── respuesta
B:             READ(x) ────────→ devuelve 0


Devuelve: READ(x) → 0

Pero si el sistema fuera linearizable, debería devolver 1. 