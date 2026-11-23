# LOAD BALANCING
Load Balancing es un método destinado a distribuir el tráfico a través de múltiples enlaces para obtener un mejor uso de cada enlace. 
 
Existen 3 métodos conocidos para balanceo de carga en RouterOS:
* ECMP (Equal Cost Multi-Path)
* Nésimo Paquete (Nth Packet)
* PCC (Per Connection Classifier)

## ECMP
1. Esquema de red

![Topología de red - ECMP](Balanceo%20de%20Carga%20-%20Topolog%C3%ADa%20de%20Red.jpg)

2. Configuración vía CLI
El Router tiene dos interfaces WAN conectadas a sus ISPs (Proveedores de servicios de Internet) con las direcciones 10.10.10.2/24 y 20.20.20.2/24.

La interfaz LAN es una interfaz bridge con el nombre "LAN10l" y la dirección IP de 192.168.10.1/24.

IP Addresses
/ ip address 
add address=192.168.10.1/24 network=192.168.10.0 broadcast=192.168.10.255 interface=LAN10
add address=10.10.10.2/24 network=10.10.10.0 broadcast=10.10.10.255 interface=ISP1
add address=20.20.20.2/24 network=20.20.20.0 broadcast=20.2.20.255 interface=ISP2 
NAT
Necesitamos agregar reglas de NAT para las direcciones de origen (src-nat) para todos los paquetes salientes. Si un paquete sale a través del ISP1, será traducido a 10.10.10.2/24, si es a través del ISP2 será traducida a 20.20.20.2/24

/ ip firewall nat 
add chain=srcnat out-interface=ether2 action=masquerade
add chain=srcnat out-interface=ether3 action=masquerade
Enrutamiento

Esta es la típica puerta de enlace ECMP (Equal Cost Multi-Path) con check-gateway. Tan pronto como no se pueda acceder a una de los Gateways, check-gateway eliminará la ruta de la lista de puertas de enlace y se realizará un Failover al otro enlace. Es decir una conmutación como consecuencia de una falla en el enlace.

/ ip route 
add dst-address=0.0.0.0/0 gateway=10.10.10.1,20.20.20.1 check-gateway=ping
Políticas de enrutamiento

Al disponer de múltiples conexiones a Internet, o múltiples Gateways, existe un problema habitual cuando llegan al router paquetes desde la red pública a través de una, otra o ambas puertas de enlace. El problema es que para los paquetes que se reciben por un ISP, podrían enviarse las respuestas a través del ISP2, de modo que esas conexiones fallarían.

Para evitarlo, necesitamos políticas de enrutamiento de esas conexiones.

/ ip firewall mangle
add chain=input in-interface=ether2 action=mark-connection new-connection-mark=ISP1_Conn
add chain=input in-interface=ether3 action=mark-connection new-connection-mark=ISP2_Conn
add chain=output connection-mark=ISP1_Conn action=mark-routing new-routing-mark=to_ISP1     
add chain=output connection-mark=ISP2_Conn action=mark-routing new-routing-mark=to_ISP2 


/ ip route
add dst-address=0.0.0.0/0 gateway=10.10.10.1 routing-mark=to_ISP1 
add dst-address=0.0.0.0/0 gateway=20.20.20.1 routing-mark=to_ISP2

### Ventajas y desventajas de ECMP

**Ventajas:**
- Configuración muy simple: una sola ruta con múltiples gateways.
- Failover automático e inmediato con `check-gateway`.
- Bajo costo de procesamiento: el router decide el enlace a nivel de ruta, sin reglas de mangle complejas (salvo el marcado para evitar asimetría con conexiones entrantes).

**Desventajas:**
- El reparto se hace por hash de IP origen/destino, no paquete a paquete: una sola conexión (por ejemplo, una descarga grande) siempre usa el mismo enlace y no se "reparte" entre ambos.
- No permite repartir en proporciones distintas si los enlaces tienen anchos de banda diferentes (por defecto reparte parejo entre los gateways listados).
- No tiene en cuenta la carga real de cada enlace en el momento: si un ISP está saturado, ECMP igual le sigue asignando conexiones nuevas.

**¿Cuándo conviene usarlo?**
Cuando los dos enlaces tienen capacidad similar y el objetivo principal es redundancia/failover simple, con la mínima complejidad de configuración posible. Es el método más básico y el punto de partida típico en MTCNA.

## NTH PACKET
1. Topologia de red:
![Topología de red - NTH Packet](Balanceo%20de%20Carga%20-%20NTH%20Packet.jpg)

2. Configuración vía CLI
Configuración vía CLI

Direcciones IP

El Router tiene dos interfaces hacia los ISPs con las direcciones de 10.10.10.2/30 y 20.20.20.2/30. La interfaz LAN10 tiene una dirección IP de 192.168.10.1/24.

/ip address
add address=192.168.10.1/24 network=192.168.10.0 broadcast=192.168.10.255 interface=LAN10
add address=10.10.10.2/30 network=10.10.10.0 broadcast=10.10.10.255 interface=ether2
add address=20.20.20.2/30 network=20.20.20.0 broadcast=20.20.20.255 interface=ether3

Balanceo de Carga

Al tener dos enlaces simétricos para el acceso a internet, marcaremos dos conexiones "ISP1" e "ISP2". Para esto cada 2 paquetes que se deban enviar a internet, el primer paquete será marcado para enviar por el ISP1 (nth=2,1) y el segungo paquete será marcado para enviar por el ISP2 (nth=2,2). 

/ip firewall mangle

add action=mark-connection chain=prerouting connection-state=new in-interface=LAN10 new-connection-mark=ISP1 nth=2,1
add action=mark-routing chain=prerouting connection-mark=ISP1 in-interface=LAN10 new-routing-mark=To_ISP1 passthrough=no

add action=mark-connection chain=prerouting connection-state=new in-interface=LAN10 new-connection-mark=ISP2 nth=2,2
add action=mark-routing chain=prerouting connection-mark=ISP2 in-interface=LAN10 new-routing-mark=To_ISP2

Entonces, lo que hace NTH es lo siguiente. Primero toma cada primer paquete que establece una nueva sesión (estado de conexión = nuevo) y lo marca con la marca de conexión "ISP1". En consecuencia, todos los paquetes sucesivos que pertenecen a la misma sesión llevarán la marca de conexión "ISP1". Tenga en cuenta que estamos pasando estos paquetes a la segunda, pues passthrough = yes. La segunda regla coloca la marca de enrutamiento (mark-routing) "ISP1" en todos los paquetes que pertenecen a la conexión "ISP1" y detiene el procesamiento de todas las demás reglas de mangle para estos paquetes en la cadena de prerounting.

Luego las reglas de NTH hacen lo mismo para la mitad restante del tráfico.  Toma cada segundo paquete que establece una nueva sesión (estado de conexión = nuevo) y lo marca con la marca de conexión "ISP2". En consecuencia, todos los paquetes sucesivos que pertenecen a la misma sesión llevarán la marca de conexión "ISP2". Tenga en cuenta que estamos pasando estos paquetes a la segunda, pues passthrough = yes. La segunda regla coloca la marca de enrutamiento (mark-routing) "ISP2" en todos los paquetes que pertenecen a la conexión "ISP2" y detiene el procesamiento de todas las demás reglas de mangle para estos paquetes en la cadena de prerounting.

Estas reglas de mangle provocan que cada nueva conexión iniciada a través del Router desde la red local sea marcada como "ISP1" o "ISP2" con marcas de mark-routing y mark-conection.

Routing

A continuación creamos una ruta para cada proveedor (routing-mark)  y activamos check-gateway  para habilitar el failover.  

/ip route
add check-gateway=ping distance=1 gateway=10.10.10.1 routing-mark=To_ISP1
add check-gateway=ping distance=1 gateway=20.20.20.1 routing-mark=To_ISP2
add check-gateway=ping distance=1 gateway=10.10.10.1
add check-gateway=ping distance=1 gateway=20.20.20.1

NAT

Necesitamos agregar reglas de NAT para las direcciones de origen (src-nat) para todos los paquetes salientes. Si un paquete sale a través del ISP1, será traducido a 10.10.10.2/24, si es a través del ISP2 será traducida a 20.20.20.2/24

/ip firewall nat
add chain=srcnat out-interface=ether2 action=masquerade
add chain=srcnat out-interface=ether3 action=masquerade

### Ventajas y desventajas de NTH

**Ventajas:**
- Reparto más granular que ECMP: permite fijar proporciones exactas entre enlaces (ej. `nth=3,1` y `nth=3,2` reparte 1/3 y 2/3), útil cuando los ISP tienen distinto ancho de banda.
- Al marcar la conexión en el primer paquete (y no cada paquete suelto), mantiene la sesión completa en el mismo enlace, siempre que el `passthrough` esté bien configurado como en el ejemplo.

**Desventajas:**
- Configuración más propensa a errores: si el marcado de conexión/`passthrough` no está bien ajustado, paquetes de una misma sesión pueden salir por gateways distintos y romper NAT o protocolos con conexiones relacionadas.
- Es una distribución "ciega" tipo round-robin: no mide la carga real de cada enlace, solo reparte en la proporción fija configurada, aunque uno de los enlaces esté más congestionado que el otro.
- Requiere más reglas de mangle que ECMP, por lo tanto más mantenimiento.

**¿Cuándo conviene usarlo?**
Cuando los enlaces tienen capacidades distintas y se necesita repartir tráfico en una proporción específica (no 50/50), priorizando control fino sobre simplicidad.

## PER CONNECTION CLASSIFIER (PCC)
1. Topología de red

![Topología de red - PCC](Balanceo%20de%20Carga%20-%20PCQ.jpg)

2. Configuración vía CLI
Configuración vía CLI

Direcciones IP

El Router tiene dos interfaces hacia los ISPs con las direcciones de 10.10.10.2/30 y 20.20.20.2/30. La interfaz LAN10 tiene una dirección IP de 192.168.10.1/24.

/ip address
add address=192.168.10.1/24 network=192.168.10.0 broadcast=192.168.10.255 interface=LAN10
add address=10.10.10.2/30 network=10.10.10.0 broadcast=10.10.10.255 interface=ether2
add address=20.20.20.2/30 network=20.20.20.0 broadcast=20.20.20.255 interface=ether3

Policy routing

Mediante la definición de políticas de enrutamiento, es posible forzar todo el tráfico a la puerta de enlace específica, incluso si el tráfico está destinado a hosts (que no correspondan a dicha puerta de enlace) desde las redes LAN conectadas. De esta forma se generará un bucle de enrutamiento y será imposible la comunicación con esos hosts. Para evitar esta situación, debemos permitir el uso de la tabla de enrutamiento predeterminada para el tráfico a las redes conectadas.

/ip firewall mangle
add chain=prerouting dst-address=10.10.10.0/30 action=accept in-interface=LAN10
add chain=prerouting dst-address=20.20.20.0/30 action=accept in-interface=LAN10

Gestión de conexiones iniciadas desde el exterior

Es necesario gestionar las conexiones iniciadas desde el exterior para que las respuestas se entreguen a través de la misma interfaz (desde la misma IP pública) por la que ingresaron. Marcaremos todas las nuevas conexiones entrantes, para recordar cuál era la interfaz. 

/ ip firewall mangle
add chain=input in-interface=ether2 action=mark-connection new-connection-mark=ISP1_Conn
add chain=input in-interface=ether3 action=mark-connection new-connection-mark=ISP2_Conn

add chain=output connection-mark=ISP1_Conn action=mark-routing new-routing-mark=to_ISP1
add chain=output connection-mark=ISP2_Conn action=mark-routing new-routing-mark=to_ISP2

Balanceo de carga

A continuación vamos a configurar el balanceo de carga, vamos a agregar dos reglas de prerouting (una para cada ISP como destino) que indentifique todo el tráfico cuyo destino no sea la red de área local (dst-address-type =! Local ) y con la ayuda del nuevo PCC, dividiremos el tráfico en dos grupos según los destinatarios de origen y destino.  

/ ip firewall mangle

add chain=prerouting in-interface=LAN10 connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses:2/0 action=mark-connection new-connection-mark=ISP1_Conn


add chain=prerouting in-interface=LAN10 connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses:2/1 action=mark-connection new-connection-mark=ISP2_Conn

Luego, debemos marcar todos los paquetes de esas conexiones con una marca adecuada. 

add chain=prerouting connection-mark=ISP1_Conn in-interface=LAN10 action=mark-routing new-routing-mark=to_ISP1 passthrough=no

add chain=prerouting connection-mark=ISP2_Conn in-interface=LAN10 action=mark-routing new-routing-mark=to_ISP2 passthrough=no

add chain=output connection-mark=ISP1_Conn action=mark-routing new-routing-mark=to_ISP1 passthrough=no

add chain=output connection-mark=ISP2_Conn action=mark-routing new-routing-mark=to_ISP2 passthrough=no

A continuación creamos una ruta para cada proveedor (routing-mark)  y activamos check-gateway  para habilitar el failover. 

/ip route
add dst-address=0.0.0.0/0 gateway=10.10.10.1 distance=1 routing-mark=to_ISP1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=20.20.20.1 distance =2 routing-mark=to_ISP2 check-gateway=ping

NAT

Necesitamos agregar reglas de NAT para las direcciones de origen (src-nat) para todos los paquetes salientes. Si un paquete sale a través del ISP1, será traducido a 10.10.10.2/24, si es a través del ISP2 será traducida a 20.20.20.2/24

/ip firewall nat
add chain=srcnat out-interface=ether2 action=masquerade
add chain=srcnat out-interface=ether3 action=masquerade

### Ventajas y desventajas de PCC

**Ventajas:**
- Es "sticky" por conexión: todos los paquetes de una misma sesión (según la combinación de IP/puerto elegida) viajan siempre por el mismo enlace, evitando que una conexión "salte" de IP pública a mitad de camino.
- Evita los problemas de NAT asimétrico que pueden darse con NTH si el marcado de conexión no está bien hecho.
- Permite elegir qué define una "sesión" (`src-address`, `dst-address`, `src-port`, `dst-port` o combinaciones), dando flexibilidad para distintos escenarios.
- Es el método recomendado por MikroTik para balanceo de múltiples WAN con NAT en redes reales.

**Desventajas:**
- Es el método con más configuración: requiere reglas extra de mangle para evitar bucles de enrutamiento con las redes directamente conectadas (bloque de "Policy routing" arriba).
- El reparto es por hash, no por carga real del enlace: con pocas conexiones concurrentes el reparto puede quedar desbalanceado (ej. si hay 2 conexiones y ambas caen en el mismo "bucket").
- Igual que los otros métodos, depende de `check-gateway` para el failover; PCC no lo da por sí solo.

**¿Cuándo conviene usarlo?**
Es la opción más recomendada en producción cuando se necesita que las sesiones de los usuarios no se corten ni cambien de IP pública en medio de la conexión: navegación web con sesiones de login, VPNs, home banking, streaming, etc. Es también el método más preguntado en el examen MTCNA junto con ECMP, por ser el que mejor resuelve el problema real de balanceo multi-WAN.

## Comparativa y recomendación general

| Método | Granularidad del reparto | Reparto proporcional (anchos de banda distintos) | Mantiene la sesión estable (sticky) | Complejidad de configuración | Caso de uso típico |
|---|---|---|---|---|---|
| ECMP | Por ruta (hash IP origen/destino) | Limitado | Sí | Baja | Failover simple entre enlaces similares |
| NTH | Por conexión nueva, round-robin fijo | Sí (proporciones configurables) | Depende del marcado de conexión | Media | Enlaces de distinta capacidad, reparto proporcional exacto |
| PCC | Por conexión (hash configurable) | Limitado | Sí, es su principal ventaja | Media-Alta | Producción, múltiples WAN con necesidad de persistencia de sesión |

**Regla práctica para el examen MTCNA:**
- Si la pregunta habla de *failover simple* entre enlaces parejos → **ECMP**.
- Si habla de *distribuir tráfico en proporciones específicas* (ej. 70/30) → **NTH**.
- Si habla de *mantener la sesión del usuario estable* o *evitar problemas de NAT asimétrico* → **PCC**.