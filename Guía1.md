**Nivel 1**: fundamentos y reconocimiento
https://www.doniacld.com/posts/2021-10-06-10-concepts-distributed-systems/

1) Es una colección de computadoras independientes que se presenta ante el usuario como si fuera una sola computadora. 
Desde una óptica operativa se puede decir que un sistema distribuído es un conjunto de procesos o nodos que:
- Se ejecutan en máquinas distintas
- Se comunican por intercambio de mensajes
- Cooperan para ofrecer una abstracción que el usuario persibe como un solo sistema. 

Por todo lo mencionado:
- la comunicación no es instantánea
- las fallas son parciales
- el conocimiento del estado global es limitado

Resumiendo, el problema se vuelve distribuído cuadno la corrección depende de coordinar componentes que no comparten memoria, reloj ni observación perfecta. 

2) Una demora suficientemente larga genera incertidumbre porque no se mantiene actualizado/al tanto al cliente del proceso que se está llevando a cabo. 
Entonces bien podría no haberse comunicado una falla O seguir esperando un resultado que lelgará dentro de X tiempo. Aunque funcionara, hbaría excedido al tiempo racional de espera
y la comunicación además sería deficiente.

3) Shazam: la aplicación captura el audio en tu teléfono, pero el procesamiento/comparación con su enorme base de datos se realiza mediante infraestructura remota.
A veces se activa la app para detectar una canción y no da una respuesta. Sin embargo, ha sucedido que horas o días después se notifica la canción buscada. 
Para esa instancia el usuario ya suponía que la detección había fallado. 
En ese caso particular, el timeout no implicó crash. 

4) La percepción de la red como un canal instantáneo y perfecto se construye sobre algunas falacias como:
    - La red es confiable: se supone que la información simepre llegará a destino
    - La latencia es cero
    - El ancho de banda es infinito
    - La red es segura
    - La topología nunca cambia
    - Sólo hay un administrador
    - El costo de transporte es cero
    - La red es homogénea
5) Para el cleinte el timeout puede resultar indistiguible de un crash. Pero un timeout significa que el cliente no recibió una respuesta dentro del tiempo esperado. Pero eso puede ocurrir por muchas razones:
   * El servidor efectivamente se cayó.
   * La red está congestionada o se perdió un mensaje.
   * La respuesta del servidor se perdió durante la transmisión.
   * El servidor está funcionando, pero está sobrecargado y tarda demasiado.
   * Hay un problema en algún nodo intermedio de la red.
En un sistema distribuido, el cliente no puede determinar con certeza qué ocurrió solamente a partir de que no recibió una respuesta. Esto se relaciona con la incertidumbre/falla parcial característica de los sistemas distribuidos.

**Nivel 2**: ejecuciones y fallas

6) Ejecución 1:
    - Cliente --> servidor: pagar $10.000
    - Servidor procesa el pago
    - Servidor --> cliente: respuesta perdida
    - Cliente espera --> timeout
    
   Ejecución 2:
    Cliente --> servidor: pagar $10.000
    La solicitud nunca llega al srevido
    Cliente espera --> time out

Ambos casos para el cliente son pagar --> esperar --> timeout
POr eso le resultan indistinguibles aunque lo que realmente suceda no es lo mismo. 
Un timeout no me permite saber si el servidor se cayó o si ocurrió otra cosa en la comunicación

7) 
- Cliente --> servidor: pagar $10.000
- Servidor procesa el pago
- Servidor --> cliente: respuesta perdida
- Cliente espera --> timeout
- Cliente --> servidor: pagar $10.000
- Servidor procesa el pago

A continuación, puede seguir sin recibir notificación o finalmente recibir una. Pero no se dará ceunta que el pago se realizó dos veces.
El nuevo riesgo es que el retry provoque la ejecución duplicada de una operación. En este caso, el primer pago puede haberse realizado correctamente, pero como la respuesta se perdió, el cliente no lo sabe y vuelve a enviar la solicitud. El servidor entonces procesa el pago nuevamente. Por lo tanto, un timeout seguido de un retry puede generar duplicación del pago.
RIESGO: DUPLICAR LA OPERACIÓN

8) Podría involucrar una caída de nodo o bien una pérdad de consistencia. 
   Cliente → Servidor: Enviar mensaje "Hola"
   Servidor recibe y guarda/procesa el mensaje.
   Servidor → Cliente: ACKnowledge / mensaje entregado
   La respuesta se pierde por un problema de red.
   Cliente no recibe el ACK y muestra "No entregado".

En tal caso, para el **servidor** el mensaje fue procesado, pero para el **cliente** el mensaje aparece como no entregado.

Si fuera el caso de una caída de nodo:

   Cliente envía mensaje.
   Nodo A recibe y procesa el mensaje.
   Nodo A replica el mensaje a Nodo B.
   Antes de enviar la confirmación al cliente, Nodo A se cae.
   El cliente no recibe la confirmación y muestra "no entregado".

9) En este caso ningún nodo se cae sino que todo siguen funcionando correctamente, pero dejan de poder comunicarse entre sí:

   - Nodo a recibe una actualización, saldo = 10.000
   - A intenta enviar la actualización a B
   - Se corta la conexión entre A y B
   - A y B siguen funcionando, pero no pueden comunicarse
   - A se qeueda con saldo 10.000
   - B sigue con el dato de 5.000
Hay una partición de la red, quedan aislados por pérdida de conectividad y tienen información diferente (lo que termina generando pérdida de consistencia).

10) Una respuesta tardía es más difícil de manejar porque genera incertidumbre sobre el estado de la operación. El cliente no puede saber si la solicitud no 
fue procesada, si fue procesada pero la respuesta se perdió, o si simplemente el servidor está tardando. En cambio, una falla explícita permite conocer 
que la operación falló y actuar en consecuencia.
Si no sabe, puede decidir reiterar la operación y se ejecutaría dos veces (como el caso de el pago doble).

**Nivel 3**: diseño y comparación

11) Tabla:

| Síntomas relevados | Causas posibles | Evidencias necesarias para distinguirlas |
|---|---|---|
| No hay mensaje de verificación | El pago llegó al servidor, pero el ACK no llegó al cliente | Revisar los logs del servidor para comprobar si recibió y procesó el pago, y los registros de comunicación para verificar si se envió el ACK |
| No se reduce el saldo de la cuenta | El pago no fue procesado; el servidor no recibió la solicitud; o hubo una inconsistencia entre nodos | Consultar el estado del pago en el servidor y comparar el saldo/estado entre las réplicas |
| La operación sigue en proceso (circulito girando) | El cliente no recibió la respuesta; el servidor está procesando la operación; o se perdió la conexión | Revisar los logs del servidor y del cliente, además del estado de la conexión, para determinar si la solicitud llegó y si fue procesada |

12) 
¿Qué se sabe?
El cliente realizó una solicitud de pago.
El cliente no recibió una respuesta dentro del tiempo esperado.
Se produjo un timeout.
El cliente, por lo tanto, no tiene confirmación de que el pago haya sido realizado.

¿Qué se infiere?
Puede haber ocurrido un problema de comunicación entre el cliente y el servidor.
Es posible que el servidor haya recibido y procesado el pago pero que la respuesta se haya perdido.
También es posible que el servidor no haya recibido la solicitud o que todavía esté procesándola.

¿Qué todavía no puede afirmarse?
No puede afirmarse que el servidor se haya caído.
No puede afirmarse que el pago no se haya realizado.
Tampoco puede afirmarse que el pago se haya realizado.
No puede saberse únicamente a partir del timeout si se perdió la solicitud, la respuesta, o si el servidor simplemente tardó demasiado.

Conclusión:
Un timeout nos da información sobre lo que observó el cliente (no recibió respuesta a tiempo), pero no nos permite conocer directamente el estado real del servidor o de la operación.

13) Propuesta de diseño

Relevamiento:
El principal problema detectado es que, ante un timeout, el cliente no puede saber si el pago fue procesado. Esto genera el riesgo de que realice un retry y el pago se ejecute dos veces, o de mostrar un saldo incorrecto.

Garantía buscada:
Garantizar que un pago no se procese más de una vez y que el saldo mostrado al usuario sea consistente con el estado confirmado de la operación.

Solución elegida:
Usar una operación idempotente, identificando cada pago con un identificador único. Si el cliente hace retry debido a un timeout, el servidor reconoce que se trata del mismo pago y no vuelve a ejecutarlo. Además, se puede confirmar la operación únicamente cuando el estado haya sido registrado de forma segura.

Si además se prioriza consistencia del saldo,se puede hacer que el sistema no permita nuevas operaciones sobre ese saldo mientras exista una operación cuyo resultado todavía no esté confirmado.

Trade-off aceptado:
Se sacrifica disponibilidad y rapidez en determinadas situaciones: el usuario puede tener que esperar o no poder realizar otra operación mientras el estado del pago no esté confirmado. A cambio, se reduce el riesgo de inconsistencias o pagos duplicados.

Por ejemplo:

* Cliente genera payment_id = 123.
* Cliente → servidor: Pagar $10.000, payment_id=123.
* Servidor procesa el pago.
* Servidor → cliente: ACK perdido.
* Cliente recibe timeout.
* Cliente hace retry con el mismo payment_id=123.
* Servidor recibe payment_id=123, busca ese ID:
* Si ya fue procesado → no vuelve a cobrar, devuelve el resultado anterior.
* Si no fue procesado → lo procesa.

Mientras el pago 123 esté en estado “pendiente de confirmación”, no permito iniciar otro pago diferente sobre esa cuenta. Si se puede reintentar la operación 123 porque está protegida por el UID.

14) COMPRAS ONLINE
P: problema --> pago remoto con timeout
F: fallas asumidas --> pérdida de respuesta, delay, retry del cliente
G: garantía requerida --> no más de un débito efectivo
S: solución elegida --> request ID único + de-duplicación + persistencia del resultado 
T: trade-off acpetado --> menor disponibilidad mientras no se pueda confirmar el estado del pago

15) PLATAFORMA DE CHAT
P: no se informa recepción
F: no llegó el mensaje | pérdida de respuesta
G: el mensaje debe llegar de A a B
S: priorizar disponibilidad. Asilar ele stado de cada mensaje
T: puede haber demoras en la actualización del estado de la entrega, pero el usuario puede seguir mensajeando con otros o con esta msima persona. 

**Nivel 4**: integración y defensa

