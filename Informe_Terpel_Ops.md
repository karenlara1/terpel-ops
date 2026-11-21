**UNIVERSIDAD DEL QUINDÍO**

Facultad de Ingeniería de Sistemas y Computación

**PROYECTO FINAL**

Nodo de soporte Terpel Ops  
Regional Occidente · Quindío

Administración de Infraestructura de TI

Estudiante: Mateo Garrido Ríos - Sebastian Torres - Jorge William Montoya - Karen Lara 
Docente: Maycol Cardenas Acevedo  
Fecha: 20 de noviembre de 2026  
Plataforma: Dokploy sobre Ubuntu Server 24.04

# 1. La organización y el problema

Organización Terpel S.A. tiene su sede en la Carrera 7 \# 75-51 de Bogotá. El informe de gestión y sostenibilidad de 2025 reporta 2.312 estaciones de combustibles líquidos en Colombia, una red latinoamericana de más de 2.300 estaciones y 32 plantas de abastecimiento en el país. La operación se organiza en siete regionales —Norte, Sur, Occidente, Centro, Sabana, Antioquia y Bucaramanga— más la Dirección General. El mismo informe muestra lubricantes Mobil, aviación, tiendas Altoque, GLP, GNV, carga eléctrica Terpel Voltex y el POS Skuba para las tiendas de conveniencia.

Este proyecto no interviene esa plataforma nacional. Responde al encargo del curso: un servidor local que centralice el sitio web, los archivos, la impresión, la hora, la autenticación de usuarios Linux y el acceso compartido. El caso de uso es un nodo de soporte en Armenia para la operación de marca en el Quindío.

El informe de Terpel no publica el mapa de departamentos por regional. Situar este piloto en la lógica de la Regional Occidente es una decisión de alcance del trabajo, coherente con la estructura territorial de la compañía, y se declara como tal.

# 2. Alcance

El nodo incluye el portal y su base, la hora común, un directorio Linux, un directorio Windows, la autenticación Linux de los técnicos, una cola de impresión y el nombre terpel.local para el laboratorio.

Quedan fuera Skuba, la facturación electrónica, SAP, el directorio corporativo nacional, la red de plantas y cualquier integración con sistemas de producción de Terpel.

El conjunto de datos del portal usa dos estaciones con dirección pública de un distribuidor de marca, y una tercera fila que es la sede académica del nodo:

| **Sitio**                    | **Ciudad** | **Dirección**            | **Papel en el piloto**               |
|------------------------------|------------|--------------------------|--------------------------------------|
| Terpel Triángulo de Arenales | Armenia    | Calle 50 No. 12-81       | EDS de combustibles líquidos         |
| Terpel Las Palmas            | Calarcá    | Avenida Colón No. 21-135 | EDS de combustibles líquidos         |
| Nodo de soporte              | Armenia    | Universidad del Quindío  | Backoffice del piloto, no es una EDS |

Los precios del portal son valores de práctica. La tarifa de surtidor de Terpel no se publica en este trabajo.

# 3. Justificación de los servicios

La siguiente tabla relaciona los servicios del nodo con su finalidad operativa y los mecanismos previstos para verificar su funcionamiento.

| **Servicio** | **Necesidad del nodo**                                                                               | **Prueba de éxito**                                                           |
|--------------|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| Portal HTTPS | Consultar la red de referencia, ajustar un precio de práctica y registrar el turno en un solo lugar. | El navegador abre https://ops.terpel.local y muestra las EDS del Quindío.     |
| Nginx        | Entregar la aplicación dentro del contenedor.                                                        | El despliegue de Dokploy queda en ejecución y el puerto del contenedor es 80. |
| Traefik      | Terminar TLS y enrutar el nombre del portal. Lo instala Dokploy.                                     | El certificado del sitio corresponde a ops.terpel.local.                      |
| PostgreSQL   | Guardar estaciones, precios y turnos, sin exponer el puerto 5432 a la LAN.                           | Un turno nuevo aparece en el portal y /api/salud responde base_de_datos ok.   |
| NTP          | Dar la misma hora al cierre de turno, al portal y a los clientes.                                    | ntpq -p en el cliente muestra la IP del servidor.                             |
| NFS          | Publicar un solo manual de cierre para los técnicos Linux.                                           | El cliente monta /mnt/terpel y lee el manual.                                 |
| NIS          | Una sola cuenta Linux para los técnicos del nodo: coord.quindio.                                     | su - coord.quindio funciona en el cliente.                                    |
| CUPS         | Una cola de impresión compartida. En el laboratorio el destino es un PDF.                            | La cola Terpel-Turnos está activa y existe un PDF en /var/spool/cups-pdf/.    |
| Samba        | Abrir la planilla de galones desde Windows.                                                          | El Explorador entra a \\IP\operaciones y ve planilla-galones.txt.             |
| DNS          | Resolver ops.terpel.local, dokploy.terpel.local y terpel-ops.terpel.local.                           | getent hosts ops.terpel.local devuelve la IP del nodo.                        |
| Dokploy      | Desplegar portal, base y proxy sin instalar Nginx en el host.                                        | El panel abre en el puerto 3000 y la aplicación figura desplegada.            |

# 4. Decisiones de diseño

- **Un servidor.** El enunciado entrega un solo equipo. El respaldo de las 02:30 y el chequeo cada cinco minutos acotan ese punto único de fallo.

- **root_squash en NFS.** El ejemplo del enunciado muestra no_root_squash. En este nodo el root del cliente no escribe en el recurso común. Quien escribe es el usuario NIS, con su propio uid.

- **Identidades separadas.** NIS autentica el ingreso al sistema operativo. La cuenta coordinador vive en la tabla usuarios de PostgreSQL. El contenedor no depende de NIS para abrir.

- **Dos directorios.** NFS y SMB no comparten el mismo árbol, para evitar el choque de bloqueos y para que cada prueba del curso quede en su protocolo.

- **Puertos RPC fijos.** mountd en 20048, statd en 32765, lockd en 32803 y ypserv en 834. Así el firewall tiene reglas concretas y no depende de puertos dinámicos.

- **PostgreSQL solo en Docker.** La aplicación se conecta por la URL interna de Dokploy. Quien consulta datos pasa por el portal.

- **TLS autofirmado.** Let's Encrypt no completa el reto HTTP contra una IP privada de laboratorio. El enunciado acepta certificado autofirmado. El PC de la demostración lo importa para mostrar el candado.

- **NFS versión 3.** El cliente monta con vers=3 para que el uid numérico de NIS llegue igual al servidor. NFSv4, en un laboratorio, suele reescribir ese usuario como nobody.

# 5. Tabla de requisitos

La dirección observada en la evidencia del laboratorio es 172.31.163.248/20, asignada dinámicamente a la interfaz eth0. La configuración propuesta contempla una dirección fija y una máscara /24; sin embargo, la captura disponible no acredita que esa configuración estática se haya aplicado. PostgreSQL se mantiene en la red interna de Docker.

| **Servicio**    | **Dirección**  | **Puerto** | **Protocolo** | **Función**                               |
|-----------------|----------------|------------|---------------|-------------------------------------------|
| Portal con TLS  | 172.31.163.248 | 443        | TCP           | Traefik termina HTTPS                     |
| Redirección web | 172.31.163.248 | 80         | TCP           | HTTP hacia HTTPS                          |
| Panel Dokploy   | 172.31.163.248 | 3000       | TCP           | Administración del PaaS, solo la LAN      |
| PostgreSQL      | red Docker     | 5432       | TCP           | Base del portal, sin publicación a la LAN |
| NTP             | 172.31.163.248 | 123        | UDP           | Hora del nodo                             |
| DNS             | 172.31.163.248 | 53         | UDP y TCP     | Zona terpel.local                         |
| NFS             | 172.31.163.248 | 2049       | TCP y UDP     | Archivos Linux                            |
| rpcbind         | 172.31.163.248 | 111        | TCP y UDP     | Registro RPC de NFS y NIS                 |
| mountd          | 172.31.163.248 | 20048      | TCP y UDP     | Montaje NFS                               |
| statd           | 172.31.163.248 | 32765      | TCP y UDP     | Recuperación de bloqueos NFS              |
| lockd           | 172.31.163.248 | 32803      | TCP y UDP     | Bloqueos NFS                              |
| NIS ypserv      | 172.31.163.248 | 834        | TCP y UDP     | Mapas de usuarios                         |
| CUPS            | 172.31.163.248 | 631        | TCP           | Impresión IPP                             |
| Samba           | 172.31.163.248 | 445        | TCP           | Archivos Windows                          |
| NetBIOS         | 172.31.163.248 | 139        | TCP           | Compatibilidad SMB                        |
| NetBIOS         | 172.31.163.248 | 137 y 138  | UDP           | Nombres antiguos                          |
| SSH             | 172.31.163.248 | 22         | TCP           | Administración                            |

# 6. Arquitectura

La figura resume el nodo. A la izquierda están los clientes del laboratorio. A la derecha, el servidor con los servicios Linux y, en la franja inferior, la plataforma Dokploy. Una petición al portal entra por Traefik, sigue a Nginx y termina en PostgreSQL. El archivo editable del diagrama, para el repositorio del curso, es diagrama.mmd.

<img src="media/image1.png" style="width:6.4in;height:3.776in" />

*Figura. Arquitectura del nodo Terpel Ops.*

# 7. Seguridad, continuidad y costo

UFW restringe el acceso a los servicios Linux de acuerdo con las reglas definidas. Los puertos publicados por Docker, incluidos 80, 443 y 3000, requieren controles adicionales porque pueden eludir las reglas de entrada de UFW. Esta condición representa una limitación de seguridad del laboratorio.

El respaldo corre a las 02:30, guarda configuración y los dos recursos compartidos, conserva siete días y, si existe /opt/terpel/backup.env con DATABASE_URL, también exporta la base. El objetivo de práctica es un RPO de 24 horas. La restauración de archivos exige CONFIRM=yes y se estima en cerca de media hora, contando la revisión manual. Cada cinco minutos el chequeo registra si NTP, NFS, Samba, CUPS, NIS, DNS y Docker siguen activos.

El costo se calcula para este nodo de laboratorio, no para la operación nacional de Terpel.

| **Concepto**                            | **Esta propuesta**                                              | **Alternativa en el mismo equipo**                                                             |
|-----------------------------------------|-----------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Servicios de hora, archivos e impresión | Ubuntu y software libre, sin licencia de estos servicios.       | Windows Server más CAL. La licencia base, en canal comercial, cuesta varios millones de pesos. |
| Portal y base                           | Dokploy, Nginx y PostgreSQL en la VM que el curso ya exige.     | Un VPS y una base administrada, con pago mensual.                                              |
| Energía                                 | La VM corre en un equipo que el laboratorio ya tiene encendido. | El mismo costo si se usa otra VM en ese equipo.                                                |
| Operación                               | Un instalador reproducible y un respaldo diario.                | Configurar cada servicio a mano, sin copia ni chequeo.                                         |

## Riesgos del piloto

| **Riesgo**                       | **Efecto**                                                 | **Tratamiento en este trabajo**                                        |
|----------------------------------|------------------------------------------------------------|------------------------------------------------------------------------|
| Caída del único servidor         | Se pierden portal, hora, archivos e impresión a la vez.    | Respaldo diario y script de restauración.                              |
| NIS en claro                     | Alguien en la LAN puede observar la autenticación.         | securenets, UFW y sin publicación a Internet.                          |
| Reloj de la VM muy desviado      | NTP se niega a sincronizar.                                | tinker panic 0 en ntp.conf.                                            |
| Puente de VirtualBox sobre Wi-Fi | La VM no recibe IP.                                        | Usar cable. Si el puente falla, red solo-anfitrión y dejar constancia. |
| Let's Encrypt en IP privada      | El certificado no se emite.                                | Certificado autofirmado generado por el instalador.                    |
| Misma clave en varias capas      | Facilita la demostración y es débil fuera del laboratorio. | Cambiarla si la VM sale de la red del curso.                           |

# 8. Máquina virtual e instalación base

El entorno de laboratorio utiliza Ubuntu Server 24.04 LTS con 2 CPU y aproximadamente 4 GB de RAM. La virtualización y la conectividad permiten ejecutar los servicios del nodo; las evidencias no permiten confirmar el uso de una interfaz en puente ni una instantánea previa.

## Copiar el proyecto a la VM

La transferencia del proyecto al servidor puede realizarse mediante SCP desde PowerShell, utilizando la dirección IP asignada a la máquina virtual.

| scp -r "C:\Users\mateo\OneDrive\Escritorio\Yo\proyecto-final" USUARIO@172.31.163.248:~/ |
|-----------------------------------------------------------------------------------------|

También se contempla la transferencia mediante carpetas compartidas del hipervisor.

## Configuración de red

La plantilla codigo/red/99-terpel-estatica.yaml.example contiene una propuesta de configuración estática. La evidencia de ip -4 addr, en cambio, muestra 172.31.163.248/20 como dirección dinámica; por ello, la configuración fija queda descrita como procedimiento propuesto, no como resultado comprobado.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>sudo cp codigo/red/99-terpel-estatica.yaml.example /etc/netplan/99-terpel-estatica.yaml</p>
<p>sudo nano /etc/netplan/99-terpel-estatica.yaml</p>
<p>sudo netplan apply</p>
<p>ip -4 addr</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

La dirección y el estado de la interfaz se consultaron mediante ip -4 addr.

Figura 01. Identificación de Ubuntu 24.04.5 LTS, dos procesadores, memoria y almacenamiento de la máquina virtual.

| <img src="media/image2.png" style="width:6.76806in;height:1.87361in" /> |
|-------------------------------------------------------------------------------------------------|

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>lsb_release -a</p>
<p>nproc</p>
<p>free -h</p>
<p>df -h /</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Figura 02. Dirección IPv4 172.31.163.248/20 en eth0, identificada como asignación dinámica.

| <img src="media/image3.png" style="width:6.76806in;height:0.54514in" /> |
|-------------------------------------------------------------------------------------------------|

## Instalador del nodo

El script deja el hostname terpel-ops, la zona America/Bogota, un swap de 2 GB, NTP, NFS, NIS, CUPS, Samba, DNS, UFW, el certificado, el cron de las 02:30 y el chequeo cada cinco minutos. Dokploy no se instala en este paso.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>cd ~/proyecto-final</p>
<p>cp codigo/scripts/terpel.env.example codigo/scripts/terpel.env</p>
<p>sudo bash codigo/scripts/instalar-servidor.sh</p>
<p>sudo bash codigo/scripts/verificar-servicios.sh</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

El instalador configura los servicios base del nodo. Su estado se comprueba mediante el script verificar-servicios.sh.

# 9. Dokploy, base de datos y portal

Como root, con los puertos 80, 443 y 3000 libres:

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>ss -ltnp | grep -E ':80|:443|:3000' || true</p>
<p>curl -fsSL https://dokploy.com/install.sh | sh</p>
<p>docker service ls</p>
<p>docker ps</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

La instalación de Dokploy contempla los servicios de administración, base de datos y proxy. La captura del panel acredita el acceso a la interfaz, aunque no demuestra por sí sola el estado de todos los contenedores.

Figura 03. Interfaz administrativa de Dokploy accesible desde el navegador.

| <img src="media/image4.png" style="width:6.76806in;height:2.94306in" /> |
|-------------------------------------------------------------------------------------------------|

## Base y aplicación

- La base de datos del portal se define en PostgreSQL con el nombre terpel_ops y el usuario terpel. Las credenciales se administran fuera del informe.

- La aplicación utiliza una URL interna de conexión a PostgreSQL con el formato postgresql://terpel:CLAVE@SERVICIO_INTERNO:5432/terpel_ops, evitando localhost dentro del contenedor.

- El portal se despliega mediante el Dockerfile de codigo/web. Nginx atiende el puerto 80 del contenedor y comunica las solicitudes a Gunicorn.

- Variables: DATABASE_URL con la URL interna; SECRET_KEY con una cadena larga; ADMIN_USER=coordinador; ADMIN_PASSWORD=\[contraseña de laboratorio\].

- El dominio previsto para el portal es ops.terpel.local, con terminación HTTPS mediante Traefik. La captura del despliegue muestra los servicios portal y terpel-db con indicadores verdes.

Figura 04. Servicios portal y terpel-db visibles en el entorno de Dokploy, ambos con indicador verde.

| <img src="media/image5.png" style="width:5.90707in;height:7.67816in" /> |
|-------------------------------------------------------------------------------------------------|

## Certificado

La solución contempla un certificado autofirmado almacenado en /etc/terpel/ssl/terpel-ops.crt y su llave privada. La configuración de Traefik permite asociarlo al dominio local. La evidencia del portal no incluye los detalles del certificado, por lo que la validación de su identidad y confianza queda pendiente de documentación.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>sudo cp /etc/terpel/ssl/terpel-ops.crt /etc/dokploy/traefik/certs/</p>
<p>sudo cp /etc/terpel/ssl/terpel-ops.key /etc/dokploy/traefik/certs/</p>
<p>sudo cp codigo/ssl/terpel-default-cert.yml /etc/dokploy/traefik/dynamic/</p>
<p>sudo docker restart dokploy-traefik</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

En Windows, la confianza en el certificado puede configurarse mediante certutil para el usuario local.

| certutil -addstore -f -user Root "C:\ruta\terpel-ops.crt" |
|-----------------------------------------------------------|

La resolución de nombres del laboratorio se apoya en el archivo hosts del cliente Windows. La llave privada del certificado permanece en el servidor.

| 172.31.163.248 ops.terpel.local dokploy.terpel.local terpel-ops.terpel.local |
|------------------------------------------------------------------------------|

El script codigo/scripts/cliente-windows.ps1 configura la resolución local, la sincronización horaria, el certificado y la unidad de red en el cliente Windows.

| .\codigo\scripts\cliente-windows.ps1 -ServerIp 172.31.163.248 -CertPath C:\ruta\terpel-ops.crt |
|------------------------------------------------------------------------------------------------|

Figura 05. Página inicial del portal Terpel Ops en el navegador; la captura no muestra los detalles del certificado TLS.

| <img src="media/image6.png" style="width:6.76806in;height:3.13542in" /> |
|-------------------------------------------------------------------------------------------------|

# 10. Contenido del portal y prueba de la base

El portal contiene las secciones Inicio, Operación y Estaciones, con los registros de Triángulo de Arenales, Las Palmas y el nodo académico. El módulo de turnos utiliza la zona horaria America/Bogota.

| curl -k https://ops.terpel.local/api/salud |
|--------------------------------------------|

El endpoint /api/salud está previsto para devolver el estado de la base de datos. No se incluye una salida visible de esta consulta en la evidencia disponible.

Figura 06. Formulario de registro de turnos y tabla de últimos turnos del portal. La imagen no muestra simultáneamente el listado de estaciones ni la respuesta de /api/salud.

| <img src="media/image7.png" style="width:6.76806in;height:3.26667in" /> |
|-------------------------------------------------------------------------------------------------|

La configuración de respaldo utiliza la URL interna de PostgreSQL, almacenada en /opt/terpel/backup.env con permisos restringidos.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>echo 'DATABASE_URL=postgresql://terpel:CLAVE@NOMBRE_INTERNO:5432/terpel_ops' | sudo tee /opt/terpel/backup.env</p>
<p>sudo chmod 600 /opt/terpel/backup.env</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

# 11. NTP

En el servidor, systemd-timesyncd queda apagado para que no dispute el puerto 123. La configuración consulta los pool públicos y, si no hay salida a Internet, ofrece el reloj local en estrato 10.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>ntpq -p</p>
<p>timedatectl</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Figura 07. Consulta de servidores NTP y estado de sincronización horaria en el servidor Ubuntu.

| <img src="media/image8.png" style="width:6.76806in;height:2.70069in" /> |
|-------------------------------------------------------------------------------------------------|

La sincronización desde una segunda máquina virtual Ubuntu se configura mediante el script cliente-linux.sh.

| sudo bash codigo/scripts/cliente-linux.sh 172.31.163.248 |
|----------------------------------------------------------|

La verificación del cliente Linux utiliza ntpq -p, herramienta que también permite identificar el servidor horario consultado.

| ntpq -p |
|---------|

En Windows, la configuración del origen horario se realiza mediante w32tm con permisos de administración.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>w32tm /config /manualpeerlist:172.31.163.248 /syncfromflags:manual /update</p>
<p>Restart-Service w32time</p>
<p>w32tm /resync /force</p>
<p>w32tm /query /status</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Figura 08. Salida de ntpq -p en el cliente Linux con referencia al servidor ops.terpel.local; no se adjunta una salida de w32tm en Windows.

| <img src="media/image9.png" style="width:6.76806in;height:0.51736in" /> |
|-------------------------------------------------------------------------------------------------|

# 12. NFS

El recurso NFS se exporta desde /srv/nfs/terpel y se monta en el cliente mediante NFSv3 en /mnt/terpel. La configuración propuesta incluye sync y root_squash; la captura acredita el montaje, pero no todas las opciones del servidor.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>sudo exportfs -v</p>
<p>showmount -e 172.31.163.248</p>
<p>findmnt /mnt/terpel</p>
<p>ls /mnt/terpel/manuales</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Figura 09. Montaje NFS del cliente Linux y listado del manual compartido en /mnt/terpel/manuales.

| <img src="media/image10.png" style="width:6.76806in;height:4.88958in" /> |
|--------------------------------------------------------------------------------------------------|

# 13. NIS

El dominio es terpel.local. El servidor es maestro y también cliente de sí mismo, para poder ejecutar ypcat. ypserv escucha en el puerto 834. nsswitch consulta primero los archivos locales y después NIS, para no perder el ingreso local si NIS falla. La cuenta de demostración es coord.quindio.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>ypcat passwd</p>
<p>ypwhich</p>
<p>su - coord.quindio</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

La cuenta de demostración es coord.quindio. La integración entre NIS y NFS se verifica mediante la propiedad de un archivo creado en el recurso compartido.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>su - coord.quindio -c 'echo prueba &gt;&gt; /mnt/terpel/turnos/prueba-nis.txt'</p>
<p>ls -l /mnt/terpel/turnos/prueba-nis.txt</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Figura 10. Consulta ypcat passwd en el cliente, donde aparece la cuenta coord.quindio; no se observa una autenticación interactiva completa.

| <img src="media/image11.png" style="width:4.92777in;height:0.61467in" /> |
|--------------------------------------------------------------------------------------------------|

Figura 11. Archivo de prueba del recurso NFS con propietario coord.quindio, según la salida de ls -l.

| <img src="media/image12.png" style="width:6.76806in;height:0.41736in" /> |
|--------------------------------------------------------------------------------------------------|

# 14. CUPS y Samba

## Impresión

La cola se llama Terpel-Turnos y escribe PDF en /var/spool/cups-pdf/. La web de administración es https://172.31.163.248:631. Entran los usuarios del grupo lpadmin.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>lpstat -p Terpel-Turnos</p>
<p>echo "cierre de turno de demostracion" | lp -d Terpel-Turnos</p>
<p>sudo ls -la /var/spool/cups-pdf</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Figura 12. Estado de la cola Terpel-Turnos y listado del directorio de salida de CUPS-PDF; no se distingue un archivo PDF generado.

| <img src="media/image13.png" style="width:6.76806in;height:1.12431in" /> |
|--------------------------------------------------------------------------------------------------|

## Archivos Windows

El recurso operaciones apunta a /srv/samba/terpel. No acepta invitado. El protocolo mínimo es SMB2. Usuario coord.quindio y la misma clave de demostración.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>testparm -s</p>
<p>smbclient //127.0.0.1/operaciones -U coord.quindio -c 'ls'</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Desde Windows se accede al recurso SMB operaciones y a la carpeta informes, que contiene la planilla compartida.

Figura 13. Recurso operaciones abierto desde el Explorador de Windows y contenido de planilla-galones.txt.

| <img src="media/image14.png" style="width:6.76806in;height:4.96181in" /> |
|--------------------------------------------------------------------------------------------------|

# 15. Firewall, respaldo, monitoreo y Compose

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>sudo ufw status verbose</p>
<p>sudo /opt/terpel/backup-terpel.sh</p>
<p>sudo ls -l /var/backups/terpel</p>
<p>cat /var/log/terpel-health.log</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Figura 14. Estado activo de UFW, reglas de acceso y directorios de respaldo generados.

| <img src="media/image15.png" style="width:6.76806in;height:5.54444in" /> |
|--------------------------------------------------------------------------------------------------|

Figura 15. Registros periódicos de salud que muestran el estado activo de los servicios supervisados.

| <img src="media/image16.png" style="width:6.76806in;height:4.28403in" /> |
|--------------------------------------------------------------------------------------------------|

La definición de Docker Compose se encuentra en codigo/web/docker-compose.yml e incluye los servicios web y db. Su comprobación se realiza de forma independiente del despliegue administrado por Dokploy.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>cd codigo/web</p>
<p>docker compose up --build -d</p>
<p>curl http://127.0.0.1:8080/api/salud</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

El despliegue con Compose constituye una alternativa de ejecución de la aplicación y no sustituye la base de datos configurada en Dokploy.

La restauración de archivos se ejecuta mediante el script restaurar-terpel.sh, previa comprobación del respaldo.

| CONFIRM=yes sudo bash /opt/terpel/restaurar-terpel.sh |
|-------------------------------------------------------|

# 16. Diagnóstico y solución de incidencias

| **Lo que se ve**                   | **Causa habitual**                                            | **Qué hacer**                                                      |
|------------------------------------|---------------------------------------------------------------|--------------------------------------------------------------------|
| Dokploy no instala                 | Algo escucha en 80, 443 o 3000.                               | ss -ltnp y apagar ese proceso. El host no debe tener Nginx.        |
| El portal reinicia                 | DATABASE_URL apunta a localhost o la base aún no está.        | Usar la URL interna de Dokploy. El contenedor reintenta un minuto. |
| El navegador no confía en el sitio | Falta importar el .crt, o Traefik sigue con otro certificado. | certutil en Windows y docker restart dokploy-traefik.              |
| ypcat vacío                        | Los mapas no se generaron.                                    | sudo make -C /var/yp y reiniciar ypserv y ypbind.                  |
| root no escribe en NFS             | root_squash está activo. Es el resultado buscado.             | Escribir como coord.quindio.                                       |
| El archivo NFS queda de nobody     | El cliente montó NFSv4.                                       | Volver a montar con -o vers=3.                                     |
| Windows no abre el recurso         | Clave Samba distinta o puerto 445 filtrado.                   | sudo smbpasswd -a coord.quindio y revisar ufw.                     |
| La hora de Windows no cambia       | PowerShell sin administrador, o UDP 123 filtrado.             | Repetir w32tm como administrador. Probar antes ntpq en Linux.      |
| UFW no cierra el puerto 3000       | Docker publica ese puerto por fuera de UFW.                   | Es el comportamiento esperado. Decirlo en la sustentación.         |

La inicialización de NIS contempla la regeneración de mapas mediante ypinit y make cuando el servicio no dispone de los datos necesarios.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p>sudo domainname terpel.local</p>
<p>sudo /usr/lib/yp/ypinit -m</p>
<p>sudo make -C /var/yp</p>
<p>sudo systemctl restart ypserv ypbind</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

La inicialización de NIS requiere confirmar el nombre del servidor maestro y generar los mapas correspondientes.

# 17. Referencias bibliográficas

- Organización Terpel S.A. Informe de gestión y sostenibilidad 2025. Cifras de EDS, regionales, sede y mención de Skuba.

- Dihego Ltda. Direcciones públicas de Terpel Triángulo de Arenales, en Armenia, y Terpel Las Palmas, en Calarcá.

- Universidad del Quindío. Enunciado del proyecto final de Administración de Infraestructura de TI, 3 de noviembre de 2026.

- Dokploy. Documentación de instalación: 2 GB de RAM, 30 GB de disco y puertos 80, 443 y 3000 libres.
