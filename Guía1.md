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