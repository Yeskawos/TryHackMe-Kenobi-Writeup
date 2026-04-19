# 🕵️‍♂️ TryHackMe: Kenobi CTF

**Autor:** Córdoba

**Fecha:** 19/04/2026

**Categoría:** Linux Enumeration & Privilege Escalation / eJPT

**Objetivo:** 10.129.171.203

## 1. Reconocimiento Inicial y Enumeración
El asalto comenzó con la verificación de conectividad y el escaneo de puertos para mapear la superficie de ataque.

* **Escaneo de Nmap y Web:** Se identificaron múltiples servicios críticos abiertos, incluyendo FTP (21), SSH (22), Web (80), RPC/NFS (111) y SMB (139/445). Una inspección visual a la web nos confirmó que el servidor estaba operativo.
* *Evidencias:*
    <img width="945" height="623" alt="ping_nmap_inicial" src="https://github.com/user-attachments/assets/cafd3346-3402-478a-af40-59c92fd3ff05" />
    <img width="933" height="863" alt="vista_web" src="https://github.com/user-attachments/assets/e3bdd98f-41d7-4675-9eb1-d92690bc26a5" />

* **Enumeración de SMB (Samba):** Dado que SMB suele ser un vector crítico de fuga de información, lanzamos `enum4linux`. Esta herramienta nos proporcionó dos piezas de información vitales:
  1. Identificamos a un usuario del sistema llamado `kenobi`.
  2. Descubrimos un recurso compartido configurado con acceso público llamado `anonymous`.
* *Evidencias:*
    <img width="939" height="950" alt="lanzamos-enum4linux" src="https://github.com/user-attachments/assets/d7301d85-1701-4950-b6c1-e107b7ce0c5e" />
    <img width="963" height="532" alt="share-enumeration-vemos-anonymous" src="https://github.com/user-attachments/assets/ddfebca3-dfe0-436d-b510-6287fc260274" />
    <img width="942" height="856" alt="user-enumeration" src="https://github.com/user-attachments/assets/98f485d6-9fbe-43c2-b682-7d314a8d2a71" />

## 2. Análisis de Fugas de Información y Planificación
Conectándonos al recurso compartido público mediante `smbclient`, encontramos un archivo de registro vital.

* **Extracción del log:** Accedimos a la carpeta `anonymous` y descargamos el archivo `log.txt`.
* *Evidencia:*
    <img width="946" height="298" alt="entramos-smbclient-anonymous" src="https://github.com/user-attachments/assets/e5888561-381d-4921-9fb0-f825032d7082" />

* **Análisis del Log:** La lectura de este archivo reveló dos vectores críticos para nuestro ataque:
  1. El servidor estaba corriendo **ProFTPD 1.3.5**.
  2. Reveló la ruta exacta donde el administrador generó y guardó la clave privada SSH del usuario Kenobi (`/home/kenobi/.ssh/id_rsa`).
* *Evidencia:*
    <img width="851" height="881" alt="cat-logtxt" src="https://github.com/user-attachments/assets/af5e607a-f337-4e35-9e21-88bc0994c94d" />

## 3. Explotación y Acceso Inicial (Initial Access)
Conociendo la versión del FTP y la ubicación de la clave SSH, buscamos vulnerabilidades públicas.

* **Identificación del Exploit:** En *Exploit-DB* encontramos que ProFTPD 1.3.5 es vulnerable a la ejecución de comandos `mod_copy`, lo que permite copiar archivos internos del servidor sin necesidad de autenticación.
* *Evidencia:*
    <img width="939" height="677" alt="busqueda-exploit-ftp-web" src="https://github.com/user-attachments/assets/994d73b1-9a72-4cb1-83a3-0a645df63ffc" />

* **Enumeración de NFS:** Para poder extraer el archivo copiado, necesitábamos un directorio público. Usando Nmap con scripts de NFS, descubrimos que la ruta `/var` entera estaba siendo compartida y exportada a toda la red.
* *Evidencia:*
    <img width="739" height="246" alt="carpeta-compartida-var" src="https://github.com/user-attachments/assets/f8c9ac80-23d7-46f3-a310-4c5b6fbfeccb" />

* **El Atraco (mod_copy):** Nos conectamos manualmente por Netcat al puerto FTP y utilizamos los comandos `SITE CPFR` (Copy From) y `SITE CPTO` (Copy To) para mover la clave SSH desde la carpeta protegida de Kenobi hasta la carpeta temporal compartida (`/var/tmp/id_rsa`).
* *Evidencia:*
    <img width="799" height="196" alt="copiamos-rsa" src="https://github.com/user-attachments/assets/81d78785-090f-4528-b2d5-8f1a459fdb2e" />


* **Obtención de la Llave y Acceso:** Montamos localmente la carpeta NFS en nuestro equipo de ataque (`/mnt/kenobiNFS`), extrajimos la llave privada, le aplicamos los permisos correctos (`chmod 600`) e iniciamos sesión exitosamente vía SSH como el usuario `kenobi`, consiguiendo la primera bandera.
* *Evidencias:*
    <img width="675" height="414" alt="mount-mnt-kenobi" src="https://github.com/user-attachments/assets/fe8d7464-35ba-442b-9ebc-88fd6c3cfd7e" />
    <img width="845" height="159" alt="cogemos-llave-damos-permisos" src="https://github.com/user-attachments/assets/c7724a12-8c77-4f7f-8f6b-1a4a85069d36" />
    <img width="821" height="686" alt="conseguimos-consola-ssh" src="https://github.com/user-attachments/assets/979d34a2-b859-4559-a630-cd5aec5895dc" />
    <img width="382" height="116" alt="flag1" src="https://github.com/user-attachments/assets/6a033b3b-8974-4bf2-a297-20270fbdc00a" />

## 4. Escalada de Privilegios (Privilege Escalation)
Operando como el usuario `kenobi`, el siguiente paso era comprometer el sistema en su totalidad (Root).

* **Búsqueda de Binarios SUID:** Ejecutamos un comando de búsqueda (`find / -perm -u=s -type f 2>/dev/null`) para localizar programas que se ejecutaran con privilegios de administrador. Destacó un archivo inusual: `/usr/bin/menu`.
* *Evidencia:*
    <img width="665" height="741" alt="buscamos-archivos-permisos-especiales" src="https://github.com/user-attachments/assets/b66229b5-db1c-4e40-bd10-3d064da10459" />

* **Ingeniería Inversa Básica:** Al ejecutar el binario, presentaba un menú interactivo. Usando el comando `strings` sobre el archivo, descubrimos que el programa invocaba comandos del sistema (como `curl`, `uname`, `ifconfig`) sin usar rutas absolutas.
* *Evidencias:*
    <img width="807" height="810" alt="comprobar-menu-hacemos-strings" src="https://github.com/user-attachments/assets/ad49db4f-35dd-43c2-a24c-7fb1f3e5a5a0" />
    <img width="557" height="380" alt="vulnerabilidad-strings-curl" src="https://github.com/user-attachments/assets/afdb596d-ea48-4a1b-9801-279f00cbe489" />

* **Ataque de Secuestro del PATH (PATH Hijacking):** Explotamos esta mala práctica de programación. Nos movimos al directorio `/tmp`, creamos un archivo malicioso llamado `curl` que en realidad abría una consola (`/bin/sh`), le dimos permisos de ejecución e inyectamos la carpeta `/tmp` al principio de la variable de entorno `$PATH`.
* *Evidencia:*
    <img width="466" height="113" alt="cambiamos-curl-nos-de-shell" src="https://github.com/user-attachments/assets/ea4f85d7-93a5-4cb5-9202-db569be0b1ea" />

* **Compromiso Total:** Al volver a ejecutar `/usr/bin/menu` y seleccionar la opción que llamaba a "curl", el sistema SUID ejecutó nuestro script falso con permisos de `root`, otorgándonos una consola interactiva de superusuario y acceso a la bandera final.
* *Evidencias:*
    <img width="708" height="172" alt="lanzamos-menu-obtenemos-shell-root" src="https://github.com/user-attachments/assets/05496e64-9e19-4755-b278-81e2a1658ec2" />
    <img width="382" height="175" alt="flag2" src="https://github.com/user-attachments/assets/71a4b245-5aa0-4ea3-82e9-5ffc32f4839c" />

## ✅ Conclusiones
La máquina Kenobi demostró la importancia crítica de aplicar el principio de mínimo privilegio y asegurar los servicios internos:
1. **Recursos Compartidos Abiertos:** Dejar recursos SMB (`anonymous`) y NFS (`/var`) abiertos permitió el descubrimiento de configuraciones internas y la exfiltración de archivos.
2. **Software Desactualizado:** La vulnerabilidad `mod_copy` en ProFTPD fue el puente necesario para mover archivos sensibles a zonas públicas sin autenticación.
3. **Malas Prácticas en Desarrollo:** La creación de binarios SUID personalizados que invocan comandos del sistema sin usar rutas absolutas (`/usr/bin/curl`) habilitó un vector clásico de escalada de privilegios mediante manipulación del entorno (PATH Hijacking).
