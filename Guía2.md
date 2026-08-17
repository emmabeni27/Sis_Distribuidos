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

Cada una de las etapas puede fallas independienemente
