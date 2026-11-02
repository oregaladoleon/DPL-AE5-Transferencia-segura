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


## Configuración

## Problemas encontrados

1. Al ejecutar lftp con:

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
sudo docker exec dpl-lab service vsftpd start
 * Starting FTP server vsftpd
   ...done.
~~~
Comprobamos:
~~~bash
udo docker exec dpl-lab ss -tlnp | grep 21
LISTEN 0      32           0.0.0.0:21        0.0.0.0:*    users:(("vsftpd",pid=879,fd=3))
~~~