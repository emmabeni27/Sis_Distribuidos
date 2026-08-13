JUEGO EN LÍNEA: Mundo virtual medieval
Servidor central y n jugadores.

**Chat**

P: no se informa recepción del mensaje (suponiendo que el el sistema lo muestra como sucede en WhatsApp, Skype o Instagram)

F: no llegó el mensaje | pérdida de respuesta. Puede ocasionarse por la pérdida de conexión a la red de alguno de los usuarios o mismo de los servidores.

G: el mensaje **debe** llegar del ususario emisor al receptor y además no se deben generar repeticiones (no puede quedar en el chat dos veces el mismo mensaje registrado)
Se busca priorizar la disponibilidad.  Cada mensaje debe funcionar como un objeto aislado. No sería admisible que por la falta de verificación del primer mensaje al usuario A
no se puedan enviar más mensajes al usuario A o bien iniciar un chat con el usuario B. 

S: pensando cada mensaje como un objeto independiente que no interfiere con el envío de los demás, podría asignársele un ID. Así si por un timeout se decide reintentar el envío, 
no se duplicará su registro. Ejemplo: si no se envió ack para el mensaje de id 123 pero su procesamiento terminó sin efectos o falló, se reintenta la operación. 
Si se detecta que el mensaje 123 sigue en procesameinto, se detiene el retry. 

T: puede haber demoras en la actualización del estado de la entrega, pero el usuario puede seguir mensajeando con otros o con esta msima persona. Se prioriza la disponibilidad pra 
hacer fluída la experiencia de usuario. El id asegurará, aunque en un plazo ligeramente más largo, la consistencia. 

**Venta compra de items (con intercambio de dinero)**

P: el dinero salió de la cuenta del comprador pero los items comprados no se encuentran en su perfil. ¿El procesamiento continúa? ¿O hubo una falla que no se notificó
y la transacción no fue atómica.

F: pérdida de respuesta (¡Compra exitosa!, delay y poterior timeout, retry del cliente --> se puede realizar un pago doble)

G: se busca garantizar la ~~atomicidad~~ [no es la atomicidad sino asegurar que sea transaccional] de la transacción. El usuario debe estar constantemente informado del estado de la transacción (evita un neuvo intento de pago).
Si ocurre una interrupción en el procesamiento de pago, se debe detener el proceso, hacer un roll back de la operación y recién leugo, un reintento.

S: ID único para la transacción y estado persistente asociado: SUCCESS, FAILED, PENDING

T: mientras se esté procesando  el pago, se interrumpe la disponibilidad para iniciar nuevas transacciones. Sin embargo, se asegura la consistencia. 


Otros puntos de análisis: logros, ganador, Batallas 1x1, compra venta de items entre jugadores