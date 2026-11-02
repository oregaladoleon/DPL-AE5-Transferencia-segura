# UT4 - Práctica evaluable, Aplicación Web con base de datos en contenedores.

> Autor: Óscar Regalado León  
> Fecha: Octubre 2026 

## Objetivo
Construir un servicio de transferencia seguro y funcional en los contenedores del **dpl-lab** y **dpl-cliente**, demostrando la creación de usuarios, directorios y permisos, transferir ficheros en modo activo y pasivo, controlar los accesos permitidos y denegados y usar FTPS y SFTP con clave pública para que las credenciales no viajen en claro.

~~~mermaid
graph TD
    subgraph Red Virtual: dpl-net
        Client["dpl-cliente\n(Cliente: lftp, scp, sftp)"]
        
        subgraph Servidor: dpl-lab
            direction BT
            FTP["Servicio vsftpd\n(Puerto 21 / FTPS Explícito)"]
            SSH["Servicio sshd\n(Puerto 22 / SFTP & SSH)"]
        end

        Client -->|FTP / FTPS (TLS)| FTP
        Client -->|SFTP / SSH (Clave Ed25519)| SSH
    end

    subgraph Almacenamiento y Permisos
        FTP -.->|Escritura: despliegue\nLectura: lector| WebDir["/srv/dpl/web\n(Propietario: despliegue:operaciones\nPermisos: 750)"]
        SSH -.->|Acceso seguro por clave| WebDir
    end
~~~
## Índice

- [Instalación](#Instalación)
- [Configuración](#configuración)
- [Comprobación](#comprobación)
- [Problemas encontrados](#problemas-encontrados)
- [Enlace al repositorio remoto](https://github.com/oregaladoleon/DPL-AE5-Transferencia-segura.git)

## Instalación
Creamos un nuevo repositorio  en la web de **Github** y lo clonamos en nuestro equipo anfitrión dentro del directorio de trabajo **~/dpl/AE4**:

~~~bash
git clone git@github.com:oregaladoleon/AE4.git AE4
~~~

Copiamos el material en el directorio del proyecto que también creamos.
