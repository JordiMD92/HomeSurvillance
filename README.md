# Home Survillance
Configurar Raspberry Pi y router Teltonika para videovigilancia usando OpenVPN

## Introduccion
Este proyecto consiste en poder visualizar multiples camaras IP remotas des de cualquier lugar. Se necesita:
- Raspberry Pi
- Teltonika RUT240
- Camara IP

### Diagrama de configuración
- IP estatica router casa: 192.168.1.1
- IP estatica raspberry pi: 192.168.1.20
- IP estatica vpn raspberry pi: 10.8.0.1
- IP estatica vpn router remoto: 10.8.0.200
- IP estatica router remoto: 192.168.2.1
- IP estatica camara1: 192.168.2.4 Puerto: 554
- IP estatica camara2: 192.168.2.5 Puerto 555
- Puerto VPN: 1194

### Motivación
- Las camaras IP están en un lugar remoto sin ADSL o FTTH, por lo que hay que usar conexion inalambrica (3g, 4g).
- Los ISP ofrecen IP's de clase A a las redes mobiles (utilizan doble NAT) por lo que utilizando un sistema DDNS no conseguiriamos conectarnos a las camaras IP. Una de las soluciones es utilizar una conexión VPN.
- Para no depender de una VPN gratuita he aprovechado una raspberry pi que estará siempre conectada en mi casa con un servidor OpenVPN entre otros servicios.
- La conexión será por RTSP. Es un protocolo extendido para la videovigilancia y casi todas las camaras lo tienen.

## Configurar Raspberry Pi
Para poner en contexto, hace falta tener una raspberry pi con Raspbian instalado.
Se utilizará PiVPN por su facilidad.
En mi raspberry tengo configurado un servidor PiHole con DHCP. Estos servicios hacen que mi configuración personal difiera respecto a una instalación limpia.
Si os interesa usar PiHole junto PiVPN recomiendo: [PiVPN and Pi-Hole](https://marcstan.net/blog/2017/06/25/PiVPN-and-Pi-hole/ "PiVPN and Pi-Hole")
1. Seguir los pasos del anterior enlace dentro del apartado **PiVPN**

2. Comprobar que tienes IP estatica. Añadir en **/etc/dhcpcd.conf**:
```bash
interface eth0
static ip_address=192.168.1.20/24
static routers=192.168.1.1
static domain_name_servers=127.0.0.1
```

3. Comprobar PUBLICDNS de PiVPN: Archivo /etc/pivpn/setupVars.conf
```bash
PUBLICDNS=nombre.dominio.com
```

4. Direccionar paquetes de las camaras a la IP y puerto correspondiente:
```bash
sudo iptables -A PREROUTING -t nat -p tcp --dport 554 -j DNAT --to-dest 10.8.0.200:554
sudo iptables -t nat -A POSTROUTING -d 10.8.0.200 -p tcp --dport 554 -j SNAT --to-source 10.8.0.1
```
> **Tener en cuenta de hacer un backup de la anterior iptables:**
> Para mantener los cambios despues de un reinicio hace falta hacer en modo root: **iptables-save > /etc/iptables/rules.v4**

5. Añadir el daemon de tu ddns para que actualize automaticamente la ip publica asociada al dominio. En mi caso [No-ip DUC](https://www.noip.com/support/knowledgebase/installing-the-linux-dynamic-update-client/ "No-ip DUC")

6. Crear el cliente vpn para el router:
```bash
pivpn add
```

7. Añadir IP estatica vpn al router remoto:
7.1. **sudo nano /etc/openvpn/server.conf** y añadir **client-config-dir /etc/openvpn/ccd**
7.2. **sudo mkdir /etc/openvpn/ccd**
7.3. **sudo chown -R pi:nogroup /etc/openvpn/ccd**
7.4. **sudo nano /etc/openvpn/ccd/XXXX**
```bash
ifconfig-push 10.8.0.200 255.255.255.0
```
> **XXXX: debe ser el mismo nombre que se ha puesto en el paso 6** 

## Configurar Teltonika RUT240
Despues de las configuraciones del wizard, comprobamos que tengamos acceso a internet. Esta configuracion te la da el propio proveedor de internet. Ademas es recomdable actualizar el firmware.

1. **Network -> Lan**
1.1. Configuramos la dirección ip para que sea 192.168.2.1.
1.2. En mi caso utilizo red lan para las 2 camaras. En opciones avanzadas hacemos check en **Use WAN port as LAN**.
1.3. Con las camaras conectadas añadimos ip's estaticas en **Static Leases**

2. **Network -> Firewall**
2.1. General Settings -> Zone Forwarding: Aceptamos forwarding de **vpn: openvpn**
2.2. Port Forwarding -> New Port Forward Rule: Añadimos:
```
Name: Nombre identificativo
Protocol: TCP+UDP
External port: 554
Internal IP: 192.168.2.5
Internal port: 554
```
2.3. Editamos esta misma regla y elegimos **Source zone: vpn: openvpn**
> Esto hay que hacerlo para cada camara con su IP y puerto. Ademas de si se quiere añadir el web cli que algunas camaras tienen

3. **Services -> VPN**
3.1. Añadimos nuevo **Client** en OpenVPN con un nombe.
3.2. Cogemos el archivo **/home/pi/ovpns/client.ovpn** del servidor y lo guardamos en algun lugar del pc.
3.2.1. Recogemos la parte interior de `<ca>`  y creamos un fichero cliente.ca con el contenido
3.2.2. Recogemos la parte interior de `<cert>`  y creamos un fichero cliente.crt con el contenido
3.2.3. Recogemos la parte interior de `<key>`  y creamos un fichero cliente.key con el contenido
3.2.4 Recogemos la parte interior de `<tls-auth>`  y creamos un fichero cliente-static.key con el contenido
3.3. Editamos el cliente y modificamos los campos acorde a la configuracion de la instalacion de PiVPN, se puede mirar en el archivo .ovpn del cliente:
```
Enable: true
TUN/TAP: TUN
Protocol: UDP
Port: 1194
LZO: false
Encryption: AES-256-CBC 256
Authentication: TLS
TLS cipher: All
Remote host/IP address: nombre.dominio.com
Resolve retry: infinete
Keep alive:
Remote network IP addres:
Remote network IP netmask:
Extra options:
HMAC authentication algorithm: SHA256
Additional HMAC authentication: true
HMAC authentication key: Añadir client-static.key
HMAC key direction: 1
Certificate authority: Añadir client.ca
Client certificate: Añadir client.crt
Client key: Añadir client.key
Private key decryption: 
```

4. **Services -> AutoReboot**
4.1. Añadir un ping reboot al gusto, con el **host to ping: 10.8.0.1**

5. **System -> Administration -> Overview**
5.1 Comprobar que **client_nombre VPN** tenga el check.

## Configurar Camara IP
En este caso las instrucciones dependen de cada marca y modelo además de si hay múltiples cámaras, y por eso comentaré la configuración básica.
- Modificar usuario y contraseña. Son dispositivos muy faciles de obtener acceso, aún estando solo accesible des de la vpn no está de más la seguridad.
- Modificar puerto RTSP para que no coincida con las demas camaras (por defecto: 554)
- (Opcional) Modificar puerto HTTP para poder configurar la camara dentro de la red VPN.

## Configurar Router
En el router hay que abrir los diferentes puertos para poder hacer la conexión.
- Puerto 1194 para la vpn a la ip de la raspberry pi (192.168.1.20)
- Puerto 554 para el rtsp de la camara a la ip de la raspberry pi (192.168.1.20)
- Todos los puertos que hagan falta para cada camara

***NO recomiendo añadir el puerto web de las camaras. Es mejor entrar estando en la red vpn.***

## Configurar IP Cam Viewer
Esta aplicacion permite la visualizacion de camaras usando RTSP.
1. Añadir camara IP
2. Nombre: Añadir nombre identificativo
3. Crear: Generic URL
4. Generic RTSP over TCP
5. URL: rtsp://nombre.dominio.com:554/XXXX<sup>1</sup>
6. User: Usuario de la camara
7. Pswd: Contraseña de la camara

Si todo está correcto, dandole a 'Test' deberia mostrar la imagen de la camara.

**XXXX<sup>1</sup>**: Esta parte de la URL te la proporciona la marca y modelo de tu camara. Si no es asi hay que hacer investigación de todas las url posibles hasta que funcione. Para comprobar las diferentes opciones recomiendo conectarse de algún modo directamente a la camara para descartar problemas con la vpn o de firewall.
Si no aparece en la documentacion recomiendo buscar en la base de datos de [iSpy](https://www.ispyconnect.com/sources.aspx "iSpy")

Por ejemplo mis camaras utilizan: 
- **rtsp://nombre.dominio.com:554/live/chX** donde X puede ser 1, 2 o 3 y definen la calidad del video.
- **rtsp://nombre.dominio.com:555/user=XXXX&password=YYYY&channel=1&stream=1.sd** donde XXXX es el usuario, YYYY la contraseña y esta camara permite conectarse con otras camaras por canales y en stream permite modificar la calidad.

# FAQ
**Importante: No usar el dns 1.1.1.1 de cloudflare en Pi-Hole y PiVPN, en su defecto usar 1.0.0.1. Algunos proveedores de internet usaban esa dirección para gestiones internas y es posible que tu router lo utilize como loopback.**

- **Por que no usar otra Raspberry pi en vez del router de Teltonika?** El usb 4G para una raspberry consume bastante, habria que usar un hub usb con alimentación. Personalmente tenia problemas y a veces el hub se quedaba desconectado (tengo el sistema montado con unas baterias y placas solares).
- **Y un HAT 4G?** Hice algunas pruebas con la Waveshare SIM7600E-H 4G HAT y era una pesadilla la configuración. Conseguí internet en 2G, nada útil.

# WIP
- Añadir modulos ZIGBEE conectados a una Raspberry Pi con lan al router remoto.
- Almacenar video en un disco duro
