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
    Servidor procesa pago