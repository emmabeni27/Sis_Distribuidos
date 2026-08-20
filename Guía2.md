**Nivel 1**: fundamentos y reconocimiento

1) RCP: Remote Procedure Call. Intenta ocultarla existencia de la red. En un sistema robusto se gestoina la red, 
no se oculta. Pero un llamado en la red tiene overhead, latencia. No podemos asimilarlo a llamar una función. 
El problema clave es lidiar con la incertidumbre, ¿la operación se ejecutó o no?

El Remote Procedure Call (RPC) intenta abstraer el hecho de que una llamada a una función o método se está ejecutando en 
otra máquina (o proceso) distinta a la que hace la llamada.

Cosas concretas que oculta/abstrae
* La comunicación de red: sockets, protocolos de transporte (TCP/HTTP), serialización y deserialización de datos.
* La ubicación física: el cliente no necesita saber en qué máquina, proceso o contenedor vive el servicio.
* El formato de los datos: los parámetros y el valor de retorno se empaquetan (marshalling) y desempaquetan (unmarshalling) automáticamente — típicamente con algo como Protocol Buffers, JSON, XML, etc.
* Los detalles del transporte: si usa sockets crudos, HTTP/2, colas de mensajes, etc.

Funciona tan bien en el caso feliz que el programador empieza a razonar sobre el código como si fuera una llamada local, y ahí es donde aparecen los problemas.

Cuando ves resultado = calcularImpuesto(monto, pais), tu cerebro aplica el modelo mental de una función local:

* Se ejecuta rápido (nanosegundos/microsegundos)
* Siempre termina: o devuelve un valor, o lanza una excepción — no hay estado intermedio raro
* Es atómica respecto a fallos: si falla, no dejó "medio hecho" nada
* La memoria es compartida: no hay que serializar/copiar datos

Ninguna de estas garantías es cierta en una llamada remota, pero la sintaxis idéntica hace que el desarrollador las asuma sin darse cuenta.

* Consecuencias concretas de esa falsa suposición
* No manejar timeouts: si asumís que la llamada "siempre vuelve", no le pones límite de tiempo, y un servidor lento puede colgar tu aplicación entera.
* No manejar fallos parciales: en una llamada local, si algo sale mal, sabés que no pasó nada. En RPC, la llamada puede fallar en la red después de que el servidor ya ejecutó la operación (por ejemplo, ya cobró la tarjeta) pero antes de que la respuesta te llegue. No sabés si se ejecutó, si no se ejecutó, o si se ejecutó dos veces.
* No pensar en idempotencia: como no esperás fallos raros, no diseñás la operación para que sea segura reintentarla.
* Ignorar el costo de latencia: hacer 50 "llamadas a función" en un loop es gratis localmente, pero si cada una es un RPC, son 50 viajes de red — esto se conoce como el problema de las "chatty APIs".
* Asumir que los datos son los mismos: en memoria compartida, un objeto pasado por referencia sigue siendo el mismo objeto. En RPC, se serializa una copia — cambios posteriores en el objeto original no se reflejan del lado remoto.

2)
* Fiabilidad: fallos parciales vs. todo-o-nada
Local: una función o se ejecuta completa y devuelve un resultado, o falla con una excepción. No hay término medio — el estado del programa antes y después es predecible.
Remota: la llamada puede fallar en cualquier punto del camino (al enviar, al ejecutar del lado del servidor, al volver la respuesta). Esto genera un problema clásico: no sabés si la operación se ejecutó o no. Si se cae la conexión después de que el servidor procesó la petición pero antes de que la respuesta te llegue, desde tu lado es indistinguible de si nunca se ejecutó.

* Latencia: nanosegundos vs. milisegundos (o más)
Local: una llamada a función cuesta unos pocos ciclos de CPU — prácticamente gratis, del orden de nanosegundos.
Remota: implica al menos un viaje de red (round-trip), con latencia de milisegundos o más, dependiendo de la distancia, congestión, etc. Esto importa mucho en diseño: hacer 100 llamadas locales en un loop no cuesta nada; hacer 100 RPCs en un loop puede tardar segundos. De ahí el problema de las "chatty APIs".

* Semántica de memoria: paso por referencia vs. copia serializada
Local: si pasás un objeto por referencia, ambas partes (llamador y función) apuntan al mismo espacio de memoria. Un cambio en un lado se ve reflejado en el otro.
Remota: los datos tienen que serializarse (marshalling) para viajar por la red, y deserializarse del otro lado. Lo que llega es siempre una copia, nunca el objeto original. Esto rompe suposiciones como "si modifico el objeto después de pasarlo, la función ve el cambio".

* Concurrencia y orden de ejecución
Local: dentro de un mismo proceso/hilo, las llamadas suelen tener un orden determinístico y previsible.
Remota: los paquetes pueden llegar desordenados, duplicarse, o perderse en la red. Esto obliga a pensar en idempotencia (¿qué pasa si la misma llamada se ejecuta dos veces por un reintento?) y en mecanismos de deduplicación — algo que nunca hace falta en una llamada local.

* Seguridad y confianza
Local: la función que llamás vive en el mismo proceso, con el mismo nivel de privilegios y sin necesidad de autenticación — confiás implícitamente en el código porque es "tuyo".
Remota: cruza un límite de confianza. Hay que autenticar quién hace la llamada, autorizar si tiene permiso para ejecutarla, y potencialmente cifrar los datos en tránsito (TLS). Esto agrega complejidad que no existe en el mundo local.

* Manejo de errores: tipos de error distintos
Local: los errores son de tu propio dominio — excepciones que vos definiste, con causas que podés inspeccionar en el mismo proceso.
Remota: aparecen categorías de error que no existen localmente — timeout, conexión rechazada, DNS no resuelve, servidor caído, versión incompatible del protocolo. El llamador tiene que distinguir entre "la función falló" y "no pude ni contactar a la función", algo que en local ni siquiera es un concepto.

* Dependencia de una red imperfecta, con fallas y puntos donde se puede caer. 

3) Etapas:

* Envío del request
* Recepción en servidor
* Ejecución
* Envío de respuesta
* Recepción en cliente

Cada una de las etapas puede fallar independienemente

4) Antes del timeout:
* El cliente envía la solicitud, el servidor la recibe y comienza a procesar. 
* Si no llega respuesta a tiempo, el cliente reintenta; para el servidor ese reintento es una posible solicitud duplicada.  (no sabe si la primera se eprdió o sigue en curso o terminó)
* Cuadno se alcanza/vence el timeout para el cliente, este considera que la lalmada falló y sigue su lógica (error, fallback, reintento) pero no sabe que hace el servidor.

Cuando el cleitne ya dejó de esperar:
* El servidor quizás sigue ejecutando y termina la operación igual. NO sabe que el cleinte ya dejó de esperar. 
* Envía rta que llega tarde. Si el cleinte ya no está esperando, la rta se descarta o pierde. 

Riesgos:
* Duplicación
* Desperdicio de trabajo (completa resutlado de una tarea que ya nadie va a usar)

Esto ilustra dos riesgos clásicos de las llamadas remotas con timeout: duplicación (si la operación no es idempotente, procesar el reintento puede ejecutar la acción dos veces) y desperdicio de trabajo (el servidor completa una tarea cuyo resultado nadie va a usar). Por eso en sistemas reales se suelen usar identificadores de solicitud (idempotency keys) y cancelación cooperativa para mitigar ambos problemas
![img_1.png](img_1.png)

5) Modelo una llamada de un cliente a un servidor remoto sobre una conexión TCP, recorriendo las capas que participan cuando algo falla — desde la aplicación hasta el cableado físico.

Aplicación (cliente RPC/HTTP y su lógica de timeout)

Respuesta ante la falla: si no llega respuesta dentro del timeout configurado, el cliente da la llamada por perdida, cierra o abandona la espera del socket, y puede reintentar o propagar un error hacia arriba.
Ayuda al diagnóstico: es la capa donde el usuario final nota el problema, y suele registrar contexto útil (qué operación, con qué parámetros, cuánto tardó).
Complica el diagnóstico: el timeout es un límite arbitrario elegido por el desarrollador, no una señal de red real. Un timeout corto puede reportar "falla" cuando en realidad la respuesta solo iba lenta, mezclando "no funcionó" con "tardó más de lo esperado".

Transporte (TCP)

Respuesta ante la falla: TCP reintenta retransmitir segmentos no confirmados con backoff exponencial: si no recibe ACK tras varios intentos, termina la conexión (RST) o la deja expirar.
Ayuda al diagnóstico: separa "el paquete se perdió una vez" (TCP lo repara solo, invisible para la aplicación) de "la conexión está realmente rota" (TCP se rinde y notifica un error de socket).
Complica el diagnóstico: los reintentos de TCP sonocultos para la aplicación, así que un problema de red intermitente puede parecer "todo funciona lento" en vez de mostrarse como errores concretos, y el tiempo que TCP gasta reintentando se suma silenciosamente al timeout de la aplicación.

Red (IP y enrutamiento)

Respuesta ante la falla: si un router no puede reenviar un paquete (destino inalcanzable, TTL agotado), puede responder con un mensaje ICMP; si no, simplemente descarta el paquete sin avisar a nadie.
Ayuda al diagnóstico: ICMP (cuando llega) da una pista concreta del punto de fallo en la ruta (herramientas como traceroute se apoyan en esto).
Complica el diagnóstico: muchos firewalls bloquean ICMP por seguridad, así que en la práctica el paquete simplemente desaparece — no hay ninguna señal explícita, solo silencio, y hay que inferir la pérdida indirectamente.

Enlace de datos (Ethernet / Wi-Fi)

Respuesta ante la falla: en Ethernet cableado, errores de trama se detectan por checksum y el frame se descarta; en Wi-Fi hay reintentos a nivel de enlace antes de darle el paquete a IP.
Ayuda al diagnóstico: los contadores de errores de la interfaz de red (CRC errors, colisiones, frames descartados) permiten distinguir un problema de cableado/interferencia de un problema más arriba en la pila.
Complica el diagnóstico: estos contadores rara vez son visibles para quien diagnostica el problema desde la aplicación; hace falta acceso a la infraestructura de red o al switch para verlos.

Física (cableado, señal, hardware de red)

Respuesta ante la falla: no hay "respuesta" lógica — un cable cortado, una interfaz caída o una señal degradada simplemente deja de transmitir bits, y esto suele reflejarse como la interfaz cayendo (link down).
Ayuda al diagnóstico: cuando existe, el evento de "link down" es una señal muy clara y binaria (o hay conexión física o no la hay).
Complica el diagnóstico: los fallos intermitentes o parciales (cable con mal contacto, interferencia) no siempre generan un evento claro, y desde las capas superiores esto se ve idéntico a cualquier otra pérdida de paquetes.

El patrón general: cada capa hacia abajo tiene menos contexto sobre "qué" se estaba haciendo pero información más precisa sobre "dónde" ocurrió el problema físico; cada capa hacia arriba tiene más contexto de negocio pero ve el fallo ya filtrado (o directamente oculto) por los reintentos de las capas inferiores. Por eso diagnosticar un timeout intermitente casi nunca se resuelve mirando una sola capa — típicamente hace falta correlacionar logs de aplicación, estadísticas de TCP (retransmisiones, RTT) y contadores de la interfaz de red para saber si el problema es de código, de congestión, o físico.

**Nivel 2**: jjecuciones y fallas