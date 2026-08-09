Guía 1 - Introducción

Sistemas centralizados, descentralizados y distribuídos

Gmail: distribuido centralizado. Hay muchos servidores distribuidos geográficamente que almacenan y procesan correos. Pero esos servidores pertenecen en su totalidad y están bajo el control de Google. No son nodos independientes que deciden entre sí cómo funciona Gmail. 

Spotify, Netflix: misma clasificación. Son muchos servidores con el contenido repartido entre ellos pero trabajan coordinadamente y toda la infraestructura y decisiones con controladas por la misma empresa.  

https://www.xataka.com/basics/servidores-nas-que-como-funcionan-que-puedes-hacer-uno
Server NAS: un servidor de este tipo es centralizado pero si todo el almacenamiento se encuentra en el mismo, NO es distribuido.
Si hubiera varios NAS cooperando, ahí sí lo podríamos ingresar a la categoría de distribuido. Pero en general la idea es que centraliza porque varios clientes dependen de un único servidor e almacenamiento. 

Home assistant: distribuido centralizado. Centralizado porque corre en un dispositivo central que mantiene el estado de los dispositivos, recibe automatizaciones, manda comandos. 
Sería distribuido si consideramos el lado de los sensores. 

IA corriendo en un data centre: depende. 
Si corre en un único servidor (nodo único) --> centralizado
Los servidores trabajan juntos pero pertenecen a la misma infraestructura y hay autoridad coordinadora  --> distribuído. Ejemplo: cada GPU hace una parte diferente. Todas pertenecen a google. 
Si no hay un servidor jefe que las controla y se coordinan por protocolo/reglas comunes--> descentralizado. 


Cluster de virtualización de computadoras: es distribuido porque los recursos de cómputo están distribuidos entre varios nodos.
Podría haber un control centralizado o no. 

Tótem de información en el medio de un parque: todo el sistema está concentrado en un único dispositivo. Es centralizado porque no hay varios nodos colaborando. 
No es distribuido porque es el único nodo prestando el servicio (asumo que es el único). No es descentralizado porque no hay varios tomando decisiones.  
-----------------------------------------------------------------------------------------
Centralizado pero distribuido: nodos de edge computing procesan datos localmente cada uno pero todo se centraliza en una empresa que administra la totalidad de los datos. Se recolecta en muchos puntos pero almacena en una única DB. 

Descentralizado: blockchain. No hay un servidor central que coordine todo, son simplemente nodos interactuando. Cada nodo mantiene una copia (o parte de la información necesaria) y se comunica con los demás.

Cajero automático

Un cajero automático es un sistema que permite a los usuarios realizar operaciones bancarias, como retirar dinero, consultar el saldo o realizar depósitos, mediante una interfaz electrónica. El cajero se comunica con los sistemas centrales del banco para verificar la identidad del usuario, consultar su cuenta y autorizar las operaciones.
Clasificación: centralizado.

Se clasifica como un sistema centralizado porque las operaciones dependen de los sistemas centrales del banco. El cajero funciona principalmente como un punto de acceso al servicio, mientras que la información de las cuentas y la validación de las operaciones se gestionan en la infraestructura central del banco.

              BANCO
        Sistema central
          /    |    \
         /     |     \
       ATM    ATM    ATM

Aunque existan muchos cajeros distribuidos geográficamente, no significa que el sistema sea distribuido: los cajeros no colaboran entre sí para realizar el procesamiento de las operaciones, sino que actúan como clientes que acceden al sistema central.
Conclusión: el cajero automático es un ejemplo de sistema centralizado, ya que el procesamiento y la información principal se encuentran centralizados en la infraestructura del banco.

-----------------------------------------------------------------------------------------
https://www.xataka.com/basics/plex-que-es-y-como-funciona

Si tuviera más de un disco con copias de la información y debiera asegurar la consistencia entre todos ellos, el sistema pasaría a involucrar múltiples nodos que deben coordinarse y replicar la información. En ese caso, el trabajo estaría repartido entre distintos nodos, por lo que sería un sistema distribuido.

Otra opción: agregar más raspberries, todas colaborando para plex. 

https://chatgpt.com/share/6a777293-03ac-83e9-b2e0-ec33fec7c7d9

Problemas de los sistemas distribuídos
https://www.baeldung.com/cs/distributed-systems-fault-failure

Problema	Caso	Solución
Compu se rompe	Un servidor puede tener varios discos RAID y uno falla.	En ese caso el sistema puede seguir funcionando y reemplazar el disco defectuoso sin apagar el servidor (hot swap). Retira físicamente el disco averiado de su bahía y reemplázalo de inmediato por uno nuevo compatible. El sistema iniciará la reconstrucción (rebuild) de forma automática o manual sin apagar el equipo
No sabe con quién comunicarse	El front no sabe permanentemente la ip donde se encuentra el back. 	Recurrir al uso de DNS (guía telefónica de IPs) o service registry para descubrir dónde está el servicio. 
No se pone de acuerdo	No logran acordar entre nodos qué valor será el próximo en registrarse en la DB.	Raft y paxos se pueden utilizar para lograr el consenso distribuido.  
Corte en comunicación	Una aplicación conectada a una VM de Azure pierde temporalmente la conexión de red con la VM. La VM continúa funcionando, pero el cliente no puede comunicarse con ella.	Esto se puede solucionar mediante timeouts, reintentos y mecanismos de reconexión/failover.
Pierden bits	Por ejemplo, está tratando de descargar una imagen pero por ruido de la transmisión los bits llegan cambiados. 	Detección de errores más retransmisión directa como checksum, CRC, códigos de detección/corrección de errores. 
		
		FCS/CRC
		Computadora A ─── Ethernet ───→ Computadora B
		                         ↓
		                    bit corrupto
		                         ↓
		                  CRC detecta error
		O con tcp…
		A ─── paquete ───→ B
		        ❌ perdido
		
		A ←── "no llegó" ── B
		A ─── retransmisión ──→ B
		
Intrusos	En canales no asegurados el atacantne puede posicionarse entre A y B haciéndolos creer que en verdad establecieron contacto. Intercambian las claves con el intruso. 	Aplicación de criptografía y autenticación. Principios de zero trust, autenticación continua, red fragmentada  y MFA. 
Bugs	No actualizan correctamente el saldo de ceunta o no contemplan la concurrencia de la reserva. 	Tests unitarios y de integración teniendo siempre la posibilidad de hacer un rollback a una versión anterior. 

