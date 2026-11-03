# UT5 - Práctica evaluable, Aplicación Web con base de datos en contenedores.

> Autor: Óscar Regalado León  
> Fecha: Noviembre 2026 

## Objetivo
Construir un servicio de transferencia seguro y funcional en los contenedores del **dpl-lab** y **dpl-cliente**, demostrando la creación de usuarios, directorios y permisos, transferir ficheros en modo activo y pasivo, controlar los accesos permitidos y denegados y usar FTPS y SFTP con clave pública para que las credenciales no viajen en claro.

~~~mermaid
graph TD
    subgraph Red Virtual: dpl-net
        Client[dpl-cliente]
        
        subgraph Servidor: dpl-lab
            FTP[vsftpd - FTP / FTPS]
            SSH[sshd - SFTP / SSH]
        end

        Client -->|FTP / FTPS| FTP
        Client -->|SFTP / SSH| SSH
    end

    subgraph Almacenamiento
        WebDir[/srv/dpl/web/]
    end

    FTP -.->|Escritura: despliegue<br/>Lectura: lector| WebDir
    SSH -.->|Acceso por clave| WebDir
~~~
## Índice

- [Instalación](#Instalación)
- [Configuración](#configuración)
- [Comprobación](#comprobación)
- [Problemas encontrados](#problemas-encontrados)
- [Enlace al repositorio remoto](https://github.com/oregaladoleon/DPL-AE5-Transferencia-segura.git)

## Instalación
1. Creamos un nuevo repositorio  en la web de **Github** y lo clonamos en nuestro equipo anfitrión dentro del directorio de trabajo **~/dpl/AE5**:

~~~bash
git clone git@github.com:oregaladoleon/DPL-AE5-Transferencia-segura.git AE5
~~~

2. Creamos el conjunto de ficheros exigidos en el guión de la práctica, además del fichero .gitignore:

~~~bash
touch README.md comandos.txt usuarios.txt permisos.txt comprobaciones.txt .gitignore
~~~

Con ayuda de gitignore.io de **Toptal** construímos el fichero **.gitignore**:

## Configuración
Los laboratorios **dpl-lab** y **dpl-cliente** tienen configuraciones heredadas de las prácticas guiadas anteriores de este tema, por lo que cuestiones como la IP asociada y estado de los contenedores ya están en activo.

3. Procedemos a crear dos usuarios: **despliegue** y **lector** y un grupo **operaciones** al que pertenecen ambos:

~~~bash
docker exec dpl-lab bash -c "
groupadd operaciones
useradd -m -s /bin/bash -G operaciones despliegue
useradd -m -s /bin/bash -G operaciones lector
echo 'despliegue:CAMBIAR_ESTO' | chpasswd
echo 'lector:CAMBIAR_ESTO' | chpasswd
"
~~~

4. Creamos el directorio del despliegue con permisos en el que el grupo sólo pueda entrar y listar pero no escribir:

~~~bash
sudo docker exec dpl-lab -c "
mkdir -p /srv/dpl/web &&
chown despliegue:operaciones /srv/dpl/web &&
chmod 750 /srv/dpl/web
"
~~~

5. Realizamos una subida con el usuario **despliegue**, subiremos el dist/ del build de Vite con **mirror -R**. Primero debemos copiar los archivos de ~/dpl/ae4/frontend/dist al directorio /tmp del contenedor **dpl-cliente**

~~~bash
docker cp ~/dpl/ae4/frontend/dist dpl-cliente:/tmp/dist

[sudo] contraseña para osrl90:          
Successfully copied 7.68kB to dpl-cliente:/tmp/dist
~~~

Luego realizamos la transferencia desde **dpl-cliente** a **dpl-lab**, debemos aceptar el certificado autofirmado en **dpl-cliente** con *ssl:verify-certificate no*

~~~bash
docker exec dpl-cliente lftp -u despliegue,CAMBIAR_ESTO 172.17.0.2 -e "set ssl:verify-certificate no; cd /srv/dpl/web; mirror -R /tmp/dist; ls; exit"

drwxr-xr-x    3 1004     1005         4096 Nov 02 22:38 dist
~~~

Aquí obtuvimos un [Problema 1](#Problema-1)

Finalmente, realizamos una descarga con el usuario **lector**:

~~~bash
docker exec dpl-cliente lftp -u lector,CAMBIAR_ESTO 172.17.0.2 -e "set ssl:verify-certificate no; cd /srv/dpl/web/dist; get index.html; ls; exit"

get: /index.html: file already exists and xfer:clobber is unset
drwxr-xr-x    2 1004     1005         4096 Nov 02 22:38 assets
-rw-r--r--    1 1004     1005          481 Nov 02 22:38 index.html
~~~

6. Comprobamos dos **accesos denegados**:

- Con el usuario **lector**:

~~~bash
docker exec dpl-cliente lftp -u lector,CAMBIAR_ESTO 172.17.0.2 -e "set ssl:verify-certificate no; cd /srv/dpl/web; put /tmp/dist/index.html; ls; exit"

put: /tmp/dist/index.html: Access failed: 553 Could not create file. (index.html)
drwxr-xr-x    3 1004     1005         4096 Nov 02 22:38 dist
~~~

- Con usuario **anónimo**:

~~~bash
sudo docker exec dpl-cliente lftp -u anonymous, 172.17.0.2 -e "ls; exit"

ls: Fatal error: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (78:0C:86:AC:69:F4:0D:9C:2C:D4:ED:B0:A6:5E:E6:E0:7C:D0:1E:3C)
~~~

- Además, para comprobar que el directorio no es accesible por otros usuarios:

~~~bash
grep -E 'despliegue|lector' /etc/passwd

lector:x:1003:1004::/home/lector:/bin/bash
despliegue:x:1004:1005::/home/despliegue:/bin/bash

groups despliegue lector

despliegue : despliegue operaciones
lector : lector operaciones

ls -ld /srv/dpl/web

drwxr-x--- 3 despliegue operaciones 4096 Nov  2 22:38 /srv/dpl/web
~~~

7. Comprobamos el acceso a **dpl-lab** en los dos modos **activo** y **pasivo**, desde el contenedor **dpl-cliente**:

~~~bash
# Modo activo (el servidor se conecta al cliente)
docker exec dpl-cliente lftp -e "set ftp:passive-mode off; set ssl:verify-certificate no; open -u despliegue,CAMBIAR_ESTO 172.17.0.2; cd /srv/dpl/web; ls; exit"

---- Resolving host address...
---- IPv6 is not supported or configured
---- 1 address found: 172.17.0.2
---- Connecting to 172.17.0.2 (172.17.0.2) port 21
<--- 220 (vsFTPd 3.0.5)
---> FEAT
<--- 211-Features:
<---  EPRT
<---  EPSV
<---  MDTM
<---  PASV
<---  PBSZ
<---  PROT
<---  REST STREAM
<---  SIZE
<---  TVFS
<--- 211 End
---> AUTH TLS
<--- 234 Proceed with negotiation.
Loaded 121 CAs
Loaded 0 CRLs
---> USER despliegue
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (78:0C:86:AC:69:F4:0D:9C:2C:D4:ED:B0:A6:5E:E6:E0:7C:D0:1E:3C)
<--- 331 Please specify the password.
---> PASS CAMBIAR_ESTO
<--- 230 Login successful.
---> PWD
<--- 257 "/home/despliegue" is the current directory
---> PBSZ 0
<--- 200 PBSZ set to 0.
---> PROT P
<--- 200 PROT now Private.
---- CWD path to be sent is `/srv/dpl/web'
---> CWD /srv/dpl/web
<--- 250 Directory successfully changed.
---> PORT 172,17,0,3,151,23
<--- 200 PORT command successful. Consider using PASV.
---> LIST
---- Accepted data connection from (172.17.0.2) port 20            # Aquí observamos que es el servidor (172.17.0.2) quien se conecta al cliente (172.17.0.3)
<--- 150 Here comes the directory listing.
Loaded 121 CAs
Loaded 0 CRLs
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (78:0C:86:AC:69:F4:0D:9C:2C:D4:ED:B0:A6:5E:E6:E0:7C:D0:1E:3C)
---- Got EOF on data connection
---- Closing data socket
<--- 226 Directory send OK.
drwxr-xr-x    3 1004     1005         4096 Nov 02 22:38 dist
---> QUIT
gnutls_record_recv: The TLS connection was non-properly terminated. Assuming EOF.
<--- 221 Goodbye.
---- Closing control socket
~~~

~~~bash
# Modo pasivo (el cliente se conecta al servidor)
docker exec dpl-cliente lftp -e "set ftp:passive-mode on; set ssl:verify-certificate no; open -u despliegue,CAMBIAR_ESTO 172.17.0.2; cd /srv/dpl/web; ls; exit"

---- Resolving host address...
---- IPv6 is not supported or configured
---- 1 address found: 172.17.0.2
---- Connecting to 172.17.0.2 (172.17.0.2) port 21
<--- 220 (vsFTPd 3.0.5)
---> FEAT
<--- 211-Features:
<---  EPRT
<---  EPSV
<---  MDTM
<---  PASV
<---  PBSZ
<---  PROT
<---  REST STREAM
<---  SIZE
<---  TVFS
<--- 211 End
---> AUTH TLS
<--- 234 Proceed with negotiation.
Loaded 121 CAs
Loaded 0 CRLs
---> USER despliegue
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (78:0C:86:AC:69:F4:0D:9C:2C:D4:ED:B0:A6:5E:E6:E0:7C:D0:1E:3C)
<--- 331 Please specify the password.
---> PASS CAMBIAR_ESTO
<--- 230 Login successful.
---> PWD
<--- 257 "/home/despliegue" is the current directory
---> PBSZ 0
<--- 200 PBSZ set to 0.
---> PROT P
<--- 200 PROT now Private.
---- CWD path to be sent is `/srv/dpl/web'
---> CWD /srv/dpl/web
<--- 250 Directory successfully changed.
---> PASV
<--- 227 Entering Passive Mode (172,17,0,2,40,42).
---- Connecting data socket to (172.17.0.2) port 10282      # Aquí observamos que es el cliente (172.17.0.3) quien se conecta al servidor (172.17.0.2)
---- Data connection established
---> LIST
<--- 150 Here comes the directory listing.
Loaded 121 CAs
Loaded 0 CRLs
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (78:0C:86:AC:69:F4:0D:9C:2C:D4:ED:B0:A6:5E:E6:E0:7C:D0:1E:3C)
---- Got EOF on data connection
---- Closing data socket
<--- 226 Directory send OK.
---> QUIT
drwxr-xr-x    3 1004     1005         4096 Nov 02 22:38 dist
gnutls_record_recv: The TLS connection was non-properly terminated. Assuming EOF.
<--- 221 Goodbye.
---- Closing control socket
~~~

Aquí obtuvimos un [Problema 2](#Problema-2)

Se ha comentado en las líneas de la salida dódne se observa quién conecta con quién.

---

8. Configuramos **FTPS explícito** (TLS sobre el puerto 21) con un certificado de laboratorio y **transferimos un fichero** con lftp forzando SSL.

Desde **dpl-lab** generamos un certificado autofirmado con **openssl** con las directivas abajo establecidas:

~~~bash
docker exec dpl-lab bash -c "apt-get install -y openssl"

docker exec dpl-lab openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/vsftpd.key -out /etc/ssl/certs/vsftpd.crt -subj '/CN=dpl.local'
~~~

Una vez generado el par de claves, añadimos a **/etc/vsftpd.conf**

~~~bash
docker exec -it dpl-lab bash

# Dentro del contenedor, aplicamos:
nano /etc/vsftpd.conf 
~~~

~~~
# fichero /etc/vsftpd.conf
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
require_ssl_reuse=NO
rsa_cert_file=/etc/ssl/certs/vsftpd.crt
rsa_private_key_file=/etc/ssl/private/vsftpd.key
~~~

Procedemos a relanzar el servicio y conectar con *set ftp:ssl-force true*, debemos especificar la ruta **cd /srv/dpl/web** porque por defecto nos deriva a la ruta **/hom/despliegue**

~~~bash
docker exec dpl-lab service vsftpd start

docker exec dpl-cliente lftp -d -u despliegue,CAMBIAR_ESTO 172.17.0.2 -e "set ftp:ssl-force true; set ssl:verify-certificate no; cd /srv/dpl/web; ls; exit"

---- Resolving host address...
---- IPv6 is not supported or configured
---- 1 address found: 172.17.0.2
---- Connecting to 172.17.0.2 (172.17.0.2) port 21
<--- 220 (vsFTPd 3.0.5)
---> FEAT
<--- 211-Features:
<---  EPRT
<---  EPSV
<---  MDTM
<---  PASV
<---  PBSZ
<---  PROT
<---  REST STREAM
<---  SIZE
<---  TVFS
<--- 211 End
---> AUTH TLS
<--- 234 Proceed with negotiation.
Loaded 121 CAs    #Aquí se verifican los certificados autofirmados
Loaded 0 CRLs
---> USER despliegue
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (78:0C:86:AC:69:F4:0D:9C:2C:D4:ED:B0:A6:5E:E6:E0:7C:D0:1E:3C) # Aquí nos lanza el aviso de que el certificado autofirmado no es de confianza.
<--- 331 Please specify the password.
---> PASS CAMBIAR_ESTO
<--- 230 Login successful.
---> PWD
<--- 257 "/home/despliegue" is the current directory
---> PBSZ 0
<--- 200 PBSZ set to 0.
---> PROT P
<--- 200 PROT now Private.
---- CWD path to be sent is `/srv/dpl/web'
---> CWD /srv/dpl/web
<--- 250 Directory successfully changed.
---> PASV
<--- 227 Entering Passive Mode (172,17,0,2,40,48).
---- Connecting data socket to (172.17.0.2) port 10288
---- Data connection established
---> LIST
<--- 150 Here comes the directory listing.
Loaded 121 CAs
Loaded 0 CRLs
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (78:0C:86:AC:69:F4:0D:9C:2C:D4:ED:B0:A6:5E:E6:E0:7C:D0:1E:3C)
---- Got EOF on data connection
---- Closing data socket
drwxr-xr-x    3 1004     1005         4096 Nov 02 22:38 dist
<--- 226 Directory send OK.
---> QUIT
gnutls_record_recv: The TLS connection was non-properly terminated. Assuming EOF.
<--- 221 Goodbye.
---- Closing control socket
~~~

9. Configuramos **SFTP con clave pública**, generamos un par de claves en **dpl-cliente**, autorizamos la pública en **dpl-lab** y transferimos un fichero con **scp** y **sftp** sin escribir contraseña.

Generamos el par de claves en **dpl-cliente**
~~~bash
docker exec dpl-cliente ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_ed25519

Generating public/private ed25519 key pair.
/root/.ssh/id_ed25519 already exists.     #De la práctica anterior.
~~~

Debemos instalar y lanzar **openssh** en **dpl-lab**:

~~~bash
docker exec dpl-lab bash -c "apt-get install -y openssh-server"

docker exec dpl-lab bash -c "mkdir -p /run/sshd && ssh-keygen -A"

docker exec dpl-lab sshd -t

docker exec -d dpl-lab /usr/sbin/sshd -D   #lanzamos en foreground

docker exec dpl-lab pgrep -a sshd

445 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups

docker exec dpl-lab ss -lntp | grep ':22'

LISTEN 0      128          0.0.0.0:22        0.0.0.0:*    users:(("sshd",pid=445,fd=3)) 
LISTEN 0      128             [::]:22           [::]:*    users:(("sshd",pid=445,fd=4))
~~~

Desde **dpl-cliente**:

~~~bash
docker exec -it dpl-cliente ssh-copy-id -o StrictHostKeyChecking=accept-new despliegue@172.17.0.2  #Autorizamos la clave pública en el servidor.

/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_ed25519.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
despliegue@172.17.0.2's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh -o 'StrictHostKeyChecking=accept-new' 'despliegue@172.17.0.2'"
and check to make sure that only the key(s) you wanted were added.
~~~
~~~bash
docker exec dpl-cliente ssh despliegue@172.17.0.2 "whoami"    #Conectamos sin contraseña. 

despliegue

docker exec dpl-cliente scp /tmp/dist/index.html despliegue@172.17.0.2:/tmp/
~~~

Al ejecutar **ssh-copy-id** ha ocurrido un [Problema 3](#Problema-3)

10. Aplicamos el endurecimiento básico de SSH (sin contraseña, sin root) y demostramos que la contraseña se rechaza pero la clave sigue funcionando.

~~~bash
docker exec dpl-lab bash -c "sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/; s/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config"

docker exec dpl-lab bash -c "grep -E 'PasswordAuthentication|PermitRootLogin' /etc/ssh/sshd_config"

PermitRootLogin no
PasswordAuthentication no
# PasswordAuthentication.  Depending on your PAM configuration,
# the setting of "PermitRootLogin prohibit-password".
# PAM authentication, then enable this but set PasswordAuthentication

docker exec dpl-lab sshd -t

docker exec dpl-lab pkill -HUP sshd
~~~

Comprobamos que la contraseña y root fallan:

~~~bash
docker exec dpl-cliente ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password -o StrictHostKeyChecking=accept-new despliegue@172.17.0.2 true

despliegue@172.17.0.2: Permission denied (publickey).

docker exec dpl-cliente ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new root@172.17.0.2 true

root@172.17.0.2: Permission denied (publickey).
~~~

## Problemas encontrados

### Problema 1

Al ejecutar lftp con:

~~~bash
docker exec dpl-cliente lftp -u despliegue,CAMBIAR_ESTO 172.17.0.2 -e "set ssl:verify-certificate no; cd /srv/dpl/web; mirror -R /tmp/dist; ls; exit"
~~~
No se ejecutaba el comando, esto era debido a que **vsftpd** no estaba activo en **dpl-lab**

~~~bash
docker exec dpl-lab pgrep -a vsftpd
#sin respuesta
~~~

Relanzamos el servicio:
~~~bash
docker exec dpl-lab service vsftpd start
 * Starting FTP server vsftpd
   ...done.
~~~
Comprobamos:
~~~bash
docker exec dpl-lab ss -tlnp | grep 21
LISTEN 0      32           0.0.0.0:21        0.0.0.0:*    users:(("vsftpd",pid=879,fd=3))
~~~

### Problema 2

Al ejecutar:

~~~bash
docker exec dpl-cliente lftp -e "set ftp:passive-mode off; set ssl:verify-certificate no; open -u despliegue,CAMBIAR_ESTO 172.17.0.2; ls; exit"


docker exec dpl-cliente lftp -e "set ftp:passive-mode on; set ssl:verify-certificate no; open -u despliegue,CAMBIAR_ESTO 172.17.0.2; ls; exit"
~~~

La terminal no nos devuelve respuesta. Hemos aplicado el modo **depuración** del comando **lftp** para identificar el error:

~~~bash
sudo docker exec dpl-cliente lftp -d -e "set ftp:passive-mode on; set ssl:verify-certificate no; open -u despliegue,CAMBIAR_ESTO 172.17.0.2; ls; exit"

---- Resolving host address...
---- IPv6 is not supported or configured
---- 1 address found: 172.17.0.2
---- Connecting to 172.17.0.2 (172.17.0.2) port 21
<--- 220 (vsFTPd 3.0.5)
---> FEAT
<--- 211-Features:
<---  EPRT
<---  EPSV
<---  MDTM
<---  PASV
<---  PBSZ
<---  PROT
<---  REST STREAM
<---  SIZE
<---  TVFS
<--- 211 End
---> AUTH TLS
<--- 234 Proceed with negotiation.
Loaded 121 CAs
Loaded 0 CRLs
---> USER despliegue
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (78:0C:86:AC:69:F4:0D:9C:2C:D4:ED:B0:A6:5E:E6:E0:7C:D0:1E:3C)
<--- 331 Please specify the password.
---> PASS CAMBIAR_ESTO
<--- 230 Login successful.
---> PWD
<--- 257 "/home/despliegue" is the current directory        #Directorio al que apunta y que está vacío.
---> PBSZ 0
<--- 200 PBSZ set to 0.
---> PROT P
<--- 200 PROT now Private.
---> PASV
<--- 227 Entering Passive Mode (172,17,0,2,40,43).
---- Connecting data socket to (172.17.0.2) port 10283
---- Data connection established
---> LIST
<--- 150 Here comes the directory listing.
Loaded 121 CAs
Loaded 0 CRLs
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (78:0C:86:AC:69:F4:0D:9C:2C:D4:ED:B0:A6:5E:E6:E0:7C:D0:1E:3C)
---- Got EOF on data connection
---- Closing data socket
<--- 226 Directory send OK.
---> QUIT
<--- 221 Goodbye.
---- Closing control socket
~~~
Al especificar en el comando la ruta (cd /srv/dpl/web) donde se alojan los ficheros /dist solucionamos el problema.

### Problema 3

Al ejecutar:

~~~bash
sudo docker exec dpl-cliente ssh-copy-id despliegue@172.17.0.2

/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_ed25519.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed

/usr/bin/ssh-copy-id: ERROR: @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
ERROR: @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
ERROR: @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
ERROR: IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
ERROR: Someone could be eavesdropping on you right now (man-in-the-middle attack)!
ERROR: It is also possible that a host key has just been changed.
ERROR: The fingerprint for the ED25519 key sent by the remote host is
ERROR: SHA256:IvVCLnEgWyAZwAAlm1XvRsNzAUSTTIllMcia1keEdR4.
ERROR: Please contact your system administrator.
ERROR: Add correct host key in /root/.ssh/known_hosts to get rid of this message.
ERROR: Offending ECDSA key in /root/.ssh/known_hosts:3
ERROR:   remove with:
ERROR:   ssh-keygen -f '/root/.ssh/known_hosts' -R '172.17.0.2'
ERROR: Host key for 172.17.0.2 has changed and you have requested strict checking.
ERROR: Host key verification failed.
~~~

Como bien indica la propia respuesta de la terminal, ya existe un fichero de la práctica anterior, procedemos a eliminarlo con el comando que proponen.

~~~bash
docker exec dpl-cliente ssh-keygen -f '/root/.ssh/known_hosts' -R '172.17.0.2'

# Host 172.17.0.2 found: line 1
# Host 172.17.0.2 found: line 2
# Host 172.17.0.2 found: line 3
/root/.ssh/known_hosts updated.
Original contents retained as /root/.ssh/known_hosts.old
~~~

Al volver a ejecutar, nos vuelve a dar error:
~~~bash
sudo docker exec dpl-cliente ssh-copy-id despliegue@172.17.0.2

/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_ed25519.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed

/usr/bin/ssh-copy-id: ERROR: Host key verification failed.
~~~

Debemos forzar con *-o StrictHostKeyChecking=accept-new*

~~~bash
docker exec -it dpl-cliente ssh-copy-id -o StrictHostKeyChecking=accept-new despliegue@172.17.0.2

/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_ed25519.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
despliegue@172.17.0.2's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh -o 'StrictHostKeyChecking=accept-new' 'despliegue@172.17.0.2'"
and check to make sure that only the key(s) you wanted were added.
~~~