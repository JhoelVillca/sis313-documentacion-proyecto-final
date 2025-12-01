# Sistema de Resiliencia Operativa y DRP (Plan de Recuperación ante Desastres)

----
## Descripción del Proyecto

### El Problema: 
En la administración de sistemas tradicional, los backups suelen fallar silenciosamente. Los scripts de copia (`cp`, `rsync`) no garantizan la consistencia si la base de datos está escribiendo en ese momento, y la recuperación manual ("DRP en papel") es lenta y propensa a errores humanos bajo presión.

> **Si un backup no ha sido probado mediante una restauración, no existe.**

### La Solución
Este proyecto implementa un **Sistema de Resiliencia Automatizada** diseñado para cumplir con los estándares de **Continuidad Operacional**. Transformamos el DRP en código ejecutable para garantizar que la recuperación sea:
1.  **Consistente:** Uso de **LVM Snapshots** para "congelar" el estado del disco en milisegundos, asegurando que la base de datos (MariaDB) nunca se copie en un estado corrupto.
2.  **Eficiente:** Implementación de **Restic** para backups incrementales con deduplicación. Solo se transmiten y almacenan los bytes que han cambiado, reduciendo el uso de red y almacenamiento.
3.  **Gestionada:** Aplicación de políticas de retención **GFS (Grandfather-Father-Son)** automáticas para mantener copias históricas sin saturar el almacenamiento.
4.  **Inmortal:** Orquestación con **Ansible** para automatizar la resurrección completa del servicio en minutos, eliminando el factor humano durante la crisis.

---

##  Arquitectura de la Solución

El sistema se distribuye en **4 Nodos Lógicos** interconectados, diseñados para simular un entorno de producción real donde los servicios (App/DB), el almacenamiento (Backups) y la gestión (Control) están desacoplados para garantizar la supervivencia de los datos incluso si los servidores principales son comprometidos.

# AQUI PONDREMOS LA TOPOLOGIA


| Nodo | Rol | Función Crítica |
| :--- | :--- | :--- |
| **VM1** | `Bóveda (Storage)` | Almacenamiento inmutable de backups (MinIO). Actúa como "caja negra" externa. |
| **VM2** | `App Node` | Servidor Web (Nginx/Apache). Representa la cara visible del negocio. |
| **VM3** | `DB Node` | Base de Datos (MariaDB) sobre lúmenes LVM. Es el activo más valioso. |
| **VM4** | `Cerebro (Control)` | Nodo de gestión desde donde Ansible ejecuta la recuperación automática. |

---

## Tecnologías Seleccionadas

### 1. Motor de Backup: Restic

* **¿Por qué?** **Restic** utiliza una arquitectura de **"Chunk-Based Deduplication"** (Deduplicación basada en fragmentos).
* **Eficiencia:** Si cambias 1 MB en una base de datos de 10 GB, Restic solo transfiere y guarda ese 1 MB. Esto cumple con el requisito de **estrategia incremental** sin la complejidad de gestionar cadenas de incrementales frágiles.
* **Seguridad:** Todo dato que sale del servidor es cifrado con **AES-256** (Cifrado simétrico fuerte con una longitud clave de 256 bits) antes de tocar la red.

### 2. Consistencia de Datos: LVM (Logical Volume Manager)
> *Requisito: Integridad en Caliente*

* **El Desafío:** Copiar los archivos de una base de datos mientras está encendida resulta en backups corruptos e inutilizables.
* **La Solución:** Utilizamos **Snapshots LVM**. Esto "congela" el sistema de archivos en el tiempo exacto en milisegundos, permitiendo a Restic copiar los datos estáticos mientras la base de datos sigue recibiendo escrituras en un espacio temporal.

### 3. Planificación: Systemd Timers

* **La Mejora:** **Systemd** maneja dependencias (ej: *"no inicies el backup si no hay red"*), reintentos automáticos si falla la conexión, y registro de logs centralizado (`journalctl`). Es vital para la **Automatización** robusta.

### 4. Orquestación DRP: Ansible
> *Requisito: Automatización de la Restauración*

* **¿Por qué?** Un Plan de Recuperación ante Desastres (DRP) documentado en papel es lento y propenso a error humano durante una crisis.
* **La Solución:** Transformamos el DRP en **Playbooks de Ansible**. Esto nos permite reconstruir el servicio, reinstalar dependencias y restaurar los datos con un solo comando, reduciendo el **RTO (Recovery Time Objective)** de horas a minutos.

---
##  Estrategia de Retención de Datos

### Perfil Producción (GFS en un entorno serio)
En un entorno empresarial real, aplicamos el esquema estándar **Grandfather-Father-Son** para cumplir con auditorías y recuperación a largo plazo:

* **Son (Diario):** `--keep-daily 7` (Mantiene 1 backup por día durante una semana).
* **Father (Semanal):** `--keep-weekly 4` (Mantiene 1 por semana durante un mes).
* **Grandfather (Mensual):** `--keep-monthly 6` (Mantiene 1 por mes durante medio año).

### Perfil Demostración (Para la demostracion para la feria)
> Debido a la naturaleza efímera del evento (ciclos de vida de minutos), hemos ajustado el "cronómetro" para operar en **Alta Frecuencia**:

* **Frecuencia de Backup:** Cada **60 segundos** (Systemd Timer).
* **Política de Retención:** `keep-last 20`.
    * *Justificación:* Esto nos permite viajar en el tiempo minuto a minuto durante la presentación, mostrando cambios inmediatos sin esperar días o semanas.



> *Esto demuestra la capacidad de Restic para gestionar el ciclo de vida de los datos sin intervención humana, escalando desde minutos (demo) hasta años (producción).*

---

##  Protocolo de Integridad 
Un riesgo crítico en sistemas automatizados es que el backup automático se ejecute *justo después* de un incidente destructivo, guardando un estado "vacío" o corrupto como el más reciente (`latest`).

**Nuestra Solución: Restauración por ID Inmutable.**
En lugar de restaurar ciegamente la etiqueta `latest`, nuestro **Menú de Control (Ansible)** permite al operador seleccionar un **Snapshot ID** específico.
1.  El sistema sigue haciendo backups (incluso del desastre), lo cual sirve como registro forense.
2.  El operador visualiza la línea de tiempo.
3.  Se selecciona el punto de restauración *previo* al incidente (T-1 minuto).

> **Resiliencia no es solo guardar datos, Tambien debemos saber a que version restaurar.**

***

# Fases de Implementación

Para garantizar la replicabilidad y el éxito del **Sistema de Resiliencia Operativa**, hemos dividido la ejecución en 9 fases estratégicas. Cada fase construye una capa de funcionalidad sobre la anterior, desde la infraestructura física hasta la interfaz de usuario final.

-----

## Fase 1: Aprovisionamiento de Infraestructura Base
**Descripción:**
Consiste en la creación y configuración inicial de los 4 Nodos Virtuales (VMs) que compondrán el sistema. Se establecen los recursos de hardware virtual (CPU, RAM, Disco) y se instala el sistema operativo base (Ubuntu Server).

**Objetivo y Aporte:**
* Establecer los cimientos del sistema.
* Segregar funciones: Separar la lógica de negocio (App/DB), el almacenamiento (Bóveda) y la gestión (Control) para evitar un "Punto Único de Fallo" (SPOF).

### Paso 1: Definición del Hardware Virtual

**📋 Descripción:**
Vamos a configurar "el chasis" de nuestras 4 máquinas virtuales en **VirtualBox**. Dado que son 4 computadoras físicas, se creara **una VM en cada laptop**.

Configura cada VM con los siguientes parámetros críticos:

| VM | Rol | RAM | CPU | Disco Principal (OS) | Discos Extra | Red |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **VM1** | `minio-vault` | 2048 MB | 2 | 25 GB (VDI Dinámico) | **+20 GB** (VDI para Backups) | NAT |
| **VM2** | `app-node` | 2048 MB | 1 | 25 GB (VDI Dinámico) | - | NAT |
| **VM3** | `db-node` | 2048 MB | 2 | 25 GB (VDI Dinámico) | **+10 GB** (VDI para LVM) | NAT |
| **VM4** | `drp-control` | 2048 MB | 1 | 25 GB (VDI Dinámico) | - | NAT |

> **Nota Crítica:** En la configuración de Red de VirtualBox, debe estar en **"NAT"**. Esto aísla la VM pero le da salida a internet usando la IP de la maquina fisica.

Establece los límites físicos de nuestros servidores. Asignar discos secundarios a la VM1 y VM3 es vital porque simula la separación profesional entre "Sistema Operativo" y "Datos Críticos". Si el OS explota, el disco de datos sobrevive.

Tener los 4 contenedores virtuales listos para recibir el sistema operativo, con el almacenamiento físico segregado correctamente para cumplir con los requisitos de LVM y Backups.

---
Recibido. Cambio de variables aceptado.
Usar usuarios distintos (`minio`, `app`, `db`, `drp`) es una práctica excelente para auditoría (sabes quién rompió qué) y refleja la realidad de un equipo distribuido.

Procedemos con el **Paso 2**. Aquí es donde el software toca el metal.

***

### Paso 2: Instalación del OS y Definición de Identidad

**Descripción:**
Instalación del sistema operativo **Ubuntu Server (LTS)** en cada una de las 4 máquinas virtuales.

**Objetivo:**
Tener 4 servidores Linux arrancando, con acceso a internet y servicio SSH activo.

Al instalar y llenar los campos de
Your name:  
Your servers name:  
Pick a username:  
choose a password:  
nosotros para esta documentacion y el proyecto trabajeremos con:
| Rol | Your name | Your servers name | Pick a username | choose a password |
|---------|---------|---------|---------|---------|
|`minio-vault`| Admin Vault | minio-vault | admin-vault | 1234 |
|`app-node`| Admin App | app-node | admin-app | 4321 |
|`db-node`| Admin DB | db-node | admin-db | 5678 |
|`drp-control`| Admin DRP | drp-control | admin-drp | 8765 |
> En un entorno real se debe tener discrecion con las contraseñas  
> En este caso usaremos estos datos, pero si se desea replicar se debe ajustar algunos comandos a sus datos.

-----

### Paso 5: agregar y verificar Discos en caso de que no se haya hecho aun. 
Verificaremos si el sistema operativo reconoce los discos secundarios (`sdb`). Si no están, apagaremos las máquinas y los "enchufaremos" virtualmente.
Confirmar que `minio-vault` tiene su bóveda de 20GB y `db-node` tiene su disco para snapshots de 10GB.

1.  **Apagar Máquina:** `sudo poweroff` (en `minio-vault` y `db-node`).
2.  **VirtualBox:**
      * Selecciona la VM.
      * Clic en **Configuración** \> **Almacenamiento**.
      * Junto a "Controlador: SATA", clic en el icono de **"Añadir Disco Duro"**.
        
      * **Crear** \> **VDI** \> **Reservado dinámicamente**.
      * **Tamaño:**
          * Para `minio-vault`: **20 GB**.
          * Para `db-node`: **10 GB**.
      * **Aceptar**.
3.  **Encender Máquina:** Inicia la VM de nuevo.
4.  **Verificar:** Corre `lsblk`. Ahora debería aparecer `sdb`.

> **Nota:** Aun no los vamos a formatearlos o montarlos todavía. De eso nos encargaremos en las fases de MinIO y LVM respectivamente.

**Una observación sobre tu tabla:**
Veo que le asignaste **1024 MB de RAM** a todas las máquinas.

  * **Advertencia:** `minio-vault` (VM1) va a correr Docker + MinIO. Con 1GB de RAM va a estar apretado. Si tu laptop física tiene memoria de sobra, súbele a **2048 MB** a la VM1 ahora que estás reiniciando. Si no, déjalo en 1GB, pero no le pidas peras al olmo si va un poco lento.

¿Confirmas que `lsblk` muestra los discos `sdb` en VM1 y VM3?
Si es afirmativo, di **"Discos Listos"** y lanzamos la **Fase 2: La Bóveda**.


------

## Fase 2: Implementación de Red de Malla (Overlay Network)
**Descripción:**
Despliegue de una red privada virtual (VPN de malla) utilizando **Tailscale/ZeroTier**. Esto crea una capa de red abstracta sobre la infraestructura física, permitiendo que las máquinas se comuniquen de forma segura y encriptada sin depender de la configuración del router local (Wi-Fi de la feria o laboratorio).

**Objetivo y Aporte:**
* **Portabilidad Total:** El sistema funciona idénticamente en el laboratorio, en una feria pública o en Internet.
* **Independencia de IP:** Uso de *MagicDNS* para resolver nombres (`db-node`, `app-node`) en lugar de depender de IPs estáticas frágiles.
* **Seguridad:** Todo el tráfico entre nodos viaja cifrado, inmune a espionaje en redes públicas.


### Paso 3: Despliegue de la Red Overlay (Tailscale)
Para trabajar este proyecto de forma remota usaremos Tailscale.
**Descripción:**
Instalación del cliente **Tailscale** en los 4 nodos. Tailscale es una VPN de malla (Mesh VPN) basada en el protocolo **WireGuard** (famoso por ser ligero y seguro).
A diferencia de las VPN tradicionales que redirigen todo el tráfico a un servidor central lento, Tailscale crea túneles encriptados punto-a-punto (P2P) entre las máquinas.

**Objetivo:**
Lograr que `minio-vault` pueda hacer ping a `db-node` usando solo su nombre, independientemente de la red física a la que estén conectados.

-----

### Procedimiento de Conexión

*Instrucciones para el equipo: Realizar en las 4 máquinas simultáneamente.*

**1. Registro en la Plataforma (Solo uno, cualquiera)**

  * Entrar a [tailscale.com](https://tailscale.com) y crear una cuenta gratuita (usar GitHub o Google).
  * Te aparece la opcion de agregar dispositivo, escojemos linux, en la parte inferior nos muestra un comando.

**2. Instalación del Agente**
En cada una de las 4 Maquinas, ejecutar el comando oficial de instalación.
*(Nota: Necesitas tener internet activo en la VM).*

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

> `curl` es una herramienta para transferir datos con URLs. Las banderas `-fsSL` le dicen: "Falla en silencio si hay error, sigue redirecciones, hazlo seguro (SSL) y no muestres la barra de carga fea".

**3. Autenticación del Nodo**
Una vez instalado, levantamos el servicio.

```bash
sudo tailscale up
```

**4. El Enlace de autenticacion**

  * La terminal mostrará un enlace largo: `https://login.tailscale.com/a/12345abcdef`.
  * **Acción:** Copia ese enlace, pégalo en el navegador de tu computadora física y autoriza la máquina con tu cuenta.
    con eso la maquina devio haberse vinculado a la cuenta, en caso de que sea la primera vez, te pedira una segunda maquina:
    ![Pide segunda maquina](Imagenes/FirstTime_SecondMachine.png)
Esto se hace para las maquinas que necesites conectar este caso solo son 4.

**5. Renombrar las Máquinas**

  * Ve al dashboar de la web de Tailscale.
  * Verás las 4 máquinas con nombres genéricos (ej. `ubuntu-2204`).
  * En caso de que no aparescan los nombres de los servidores
  * Haz clic en los 3 puntos (...) y cambiarles el nombre para que coincidan con nuestra arquitectura:
      * `minio-vault`
      * `app-node`
      * `db-node`
      * `drp-control`
  * Esta opcion normalmente viene ya activado, pero en caso de que no ve a tailscale y  DNS->MagicDNS Y Activa la opción.

-----

### Verificación

Desde cualquier maquina por ejemplo `drp-control`, intenta contactar a la base de datos por su nombre:

```bash
ping db-node -c 4
```
> Deveria poder hacer ping a cualquiera de las maquinas desde cualquiera de las maquinas en este punto.
> y con eso tenemos nuestro DNS externo (centralizado) funcionando
-----

### Paso 4: Redundancia de Nombres (Configuración de `/etc/hosts`) (No nesesario)
Esta es otra forma de asignar dns, solo que esta ves de forma local, que lo usaremos por si nuestro DNS externo falla por alguna razon, es completamente *opcional*. 
En Producción: siempre se debe usar DNS (centralizado) para evitar errores administrativos.  
En Desarrollo/Pruebas: Usa /etc/hosts (local) para simular conexiones de forma rápida y segura, en este caso como una segunda opcion.

Editaremos el archivo local de resolución de nombres (`/etc/hosts`) en cada una de las 4 máquinas. Esto garantiza que el sistema sepa quién es `db-node` incluso si Tailscale MagicDNS falla.
Además, este es el mismo archivo que editaríamos de emergencia si tuviéramos que cambiar a una red física (Cable/Switch) sin internet.

**Aporte al Proyecto:**

  * **Resistencia a Fallos DNS:** Si el servidor dns muere, nuestro archivo local tiene la verdad absoluta.
  * **Velocidad:** Consultar un archivo local es más rápido que preguntar a la red.

**Obtener el Mapa de Direcciones**
Necesitas saber la IP de Tailscale de cada una de tus 4 máquinas (empiezan por `100.x.y.z`).
Ejecuta esto en cada máquina o míralo en tu panel web de Tailscale:

```bash
tailscale ip -4
```
> `tailscale` Es la herramienta que estamos usando  
> `ip` con esto le decimos a tailscale que nos muestre la ip  
> `-4` con esto le decimos que queremos ver solo el ipv4, si ponemos `-6` nos mostrara el ipv6.    

una ves veas las ip de cada maquina en mi caso: 

  * `minio-vault`: `100.73.190.14`
  * `app-node`: `100.105.30.93`
  * `db-node`: `100.109.57.70`
  * `drp-control`: `100.66.35.17`

**2. Editar el Archivo Hosts**
Debes hacer esto **EN LAS 4 MÁQUINAS**. El archivo debe quedar idéntico en todas.

Abre el archivo:

```bash
sudo nano /etc/hosts
```
> Es el archivo que sirve para resolver nombres de dominio de manera Local  

**3. Agregar DNS Locales**
Al final del archivo, pega las 4 direcciones, con el formato de [ipv4]   [El nombre que le daremos a la ip, en muestro caso usaremos los nombres de los servidores]
```text
100.73.190.14   minio-vault
100.105.30.93   app-node
100.109.57.70   db-node
100.66.35.17    drp-control
```

**4. Verificación**
desactivando el MagicDNS de tailscale
Desde `drp-control` o cualquier vm, intenta hacer ping usando el nombre:
```bash
ping db-node -c 2
```
si funciona, entonces esta bien, puedes volver a activar MagicDNS o no, como se desee.
En caso de que por alguna razon cambien de direccion de red solo se debe ajustar en `/etc/hosts`

-----

### En caso de que se requiera no usar Tailscale, o el internet se corta.

Como configuramos las VMs en modo **NAT** (para que funcionen con Tailscale), si ya no podemos usar el tunel, las máquinas quedan aisladas
En ese caso seguiremos esto para poder trabajar sin internet ni Tailscale, y se usara la red local LAN (Compartir la misma red de wifi, ya sea con celular o un router):
1.  **Apagar VMs:** `sudo poweroff`.
2.  **Cambio Físico:** En VirtualBox de cada maquina, cambiar Red de **NAT** a **Adaptador Puente**.
3.  **Encender VMs:** Ahora tomarán IPs del router local (ej. `192.168.1.x`).
4.  **Actualizar Hosts:** Entras a `/etc/hosts` de nuevo y cambias las IPs `100.x` por las nuevas `192.168.x`.
5.  **Resultado:** Esto deberia permitir la coneccion entre los servidores, sin tener que ajustar todos los script (razon por la que usamos DNS).


-----

## Fase 3: Despliegue de la Bóveda Inmutable (Object Storage)
**Descripción:**
Instalación y configuración de **MinIO** (compatible con Amazon S3) en un entorno contenerizado (Docker). Se configuran las políticas de acceso, usuarios de servicio y persistencia de datos en disco.

**Objetivo y Aporte:**
* Crear un repositorio centralizado y desacoplado para los backups.
* Simular una arquitectura de nube real (Cloud-Native) en un entorno local.
* Garantizar que si los servidores de aplicación son destruidos, los datos de respaldo permanezcan intactos en un "búnker" aislado.


-----

### Paso 1: Preparación del Almacenamiento Físico (minio-vault)

**Ubicación:** Ejecutar EXCLUSIVAMENTE en **La maquina `minio-vault`**.
**Objetivo:** Formatear el disco extra de 20GB (`/dev/sdb`) y montarlo permanentemente en `/mnt/data`. Esto garantiza que los backups vivan en un disco separado del sistema operativo.

#### 1\. Identificar el disco agregado

Verificamos que el sistema detecte el disco de 20GB.

```bash
lsblk
```

> Esto Lista los dispositivos de bloque. Deberías ver `sdb` con 20G de tamaño y sin particiones, si es asi esta bien.

#### 2\. Formatear el disco (Crear sistema de archivos)

Le daremos formato **ext4**, el estándar de Linux.

```bash
sudo mkfs.ext4 /dev/sdb
```

> `mkfs`  convierte el disco bruto en algo donde se pueden guardar archivos.
> `ext4` es el estándar actual y soporta discos grandes, journaling avanzado, menos fragmentación, existen otras opciones como ext2, ext3, XFS, Btrfs y ZFS.
> Basicamente dice formatea el disco /dev/sdb y prepáralo para guardar archivos usando el sistema de archivos ext4.
 
#### 3\. Crear el punto de montaje

Creamos la carpeta donde "enchufaremos" este disco.

```bash
sudo mkdir -p /mnt/data
```
> mkdir crea un directorio, -p lo usamos por si ese directorio ya existe, entonces crea lo que falta, /mnt/data es el directorio.
> **Explicación:** `/mnt/data` será la puerta de entrada. Todo lo que guardemos aquí irá físicamente al disco de 20GB.
> Basicamente dice Crea la carpeta llamada data dentro del directorio /mnt. Si ya existe, no pasa nada.

#### 4\. Montaje manual (Prueba)

Conectamos el disco a la carpeta.

```bash
sudo mount /dev/sdb /mnt/data
```
> mount sirve para conectar un dispositivo de almacenamiento a un directorio. en este caso conecta el disco `/dev/sdb` al directorio `/mnt/data`.
> **Explicación:** Enlaza el dispositivo físico `/dev/sdb` con el directorio `/mnt/data`.

#### 5\. Montaje persistente (Para reinicios)

Si reinicias ahora, el disco se desconectará. necesitamos que se monte el disco solito cada vez que lo reiniciamos, Editamos `/etc/fstab` para que se monte solo al arrancar.

```bash
sudo nano /etc/fstab
```
> Ese es el archivo de montajes automaticos.
Al final del archivo agregamos:
```bash
/dev/sdb   /mnt/data   ext4   defaults   0   0
```
> /dev/sdb es el disco.
> /mnt/data es el directorio o el punto de montaje.
> ext4 es el sistema de archivos.
> 0 0 es para no usar dump y no revisar automáticamente este disco con fsck al arrancar, el primero desactiva copias automaticas con dump, y el segundo 0 desactiva el chequeo automatico con fsck, fsck sirve para chequear y reparar el sistema de archivos, no lo usamos por que es lento, tarda unos minutos en revisar un disco grande.
> **Explicación:** Escribe una línea en la "tabla de sistemas de archivos" (`fstab`) instruyendo al kernel que monte `sdb` en `/mnt/data` en cada arranque.

#### 6\. Asignar permisos para MinIO

MinIO es un software seguro y no corre como "root". Corre con el usuario ID `1001`. Debemos regalarle esta carpeta a ese ID para que pueda escribir en ella.

```bash
sudo chown -R 1001:1001 /mnt/data
```
> chown -> cambiar porpiertario.
> -R para seleccionar la carpeta y sus subcarpetas y archivos.
> 1001:1001 quien sera nuestro nuevo propietario esta con el formato de Usuario:Grupo, 1001 es el UID de MinIO
> **Explicación:** `chown` (Change Owner) cambia el dueño de la carpeta. `1001:1001` es el Usuario:Grupo que usará el contenedor Docker de MinIO.
> si bien aun no tenemos instalado aun minIO, lo ponemos para que cuando usemos minio ya pueda escribir en esa carpeta.
-----

**Verificación:**
Ejecuta `df -h | grep /mnt/data`.
> df -> muestra el espacio libre.  
> -h convierte los tamanios a KB, MB, GB para que sea facil leer.  
> grep /mnt/data filtra la salida y muestra solo las líneas que contienen /mnt/data.  
> basicamente dice Muéstrame el espacio disponible y usado en el disco que está montado en /mnt/data
Deberías ver una línea que dice `Size: 20G` (aprox).

-----  

### Paso 2: Despliegue del Motor (Docker + MinIO)

**Ubicación:** Ejecutar EXCLUSIVAMENTE en **VM1 (`minio-vault`)**.
**Objetivo:** Instalar el motor de contenedores Docker y desplegar la instancia de MinIO conectada al almacenamiento físico que preparamos en el paso anterior.

#### 1\. Instalación de Docker

Usaremos la versión mantenida por los repositorios oficiales de Ubuntu (`docker.io`) por su estabilidad y facilidad de instalación.

```bash
sudo apt update && sudo apt install docker.io -y
```

> **Explicación:** Actualiza la lista de paquetes para asegurarnos de bajar la última versión disponible e instala el motor de Docker. La bandera `-y` responde "Yes" automáticamente a las preguntas de confirmación.

#### 2\. Permisos de Usuario (Evitar `sudo` constante)

Por defecto, Docker solo obedece al usuario `root`. Para que nuestro usuario (`admin-vault` o el que estés usando) pueda dar órdenes sin escribir `sudo` a cada rato, lo agregamos al grupo privilegiado.

```bash
sudo usermod -aG docker $USER
```
> `usermod`: Modificar las configuraciones de un usuario.  
> `-aG`: **A**ppend (Agregar) al **G**rupo.  
> `docker`: El nombre del grupo especial.  
> `$USER`: Variable de entorno que se reemplaza automáticamente por tu usuario actual.  
> Basicamente dice Que agregue a nuestro usuario al grupo de personas que tienen el control total de docker, por defecto biene que solo sea el usuario root, pero con esto el usuario actual tendra tambien los permisos.


** Acción necesaria:** Para que este cambio surta efecto, debes **cerrar sesión y volver a entrar** (logout/login) o ejecutar, si es por ssh con exit y luego volver a entrar:

```bash
newgrp docker
```

#### 3\. Despliegue del Contenedor MinIO

Este es el comando principal que levanta el servidor. Copia y pega el bloque completo.

```bash
docker run -dt \
  -p 9000:9000 -p 9001:9001 \
  --name minio \
  --restart always \
  -v /mnt/data:/data \
  -e "MINIO_ROOT_USER=admin" \
  -e "MINIO_ROOT_PASSWORD=SuperSecretKey123" \
  minio/minio server /data --console-address ":9001"
```

> **Explicacion del Comando:**
>  
> `docker run -dt`: Crea e inicia un contenedor en modo **d**etached (segundo plano) y asigna una **t**ty (terminal virtual).  
> `-p 9000:9000`: Expone el puerto API (donde se envían los datos).  
> `-p 9001:9001`: Expone el puerto de la Consola Web (donde tú ves los archivos).
> `--name minio`: Es el nombre que le daremos al contenedor "minio".  
> `--restart always`: Para que docker arracara solo. Si el servidor se apaga por corte de luz o error, Docker revivirá este servicio automáticamente al arrancar.  
> `-v /mnt/data:/data`: Conecta nuestra carpeta del disco duro físico (`/mnt/data`) con la carpeta interna del contenedor (`/data`). Si borras el contenedor, los datos siguen seguros en el disco.  
> `-e -e "MINIO_ROOT_USER=admin"` y `-e "MINIO_ROOT_PASSWORD=SuperSecretKey123"
`: Definimos la clave y el usuario, en un entorno real se debe usar claves mas complejas.  
> `minio/minio server /data`: La imagen a usar y Arranca en modo servidor y usa la carpeta /data como almacenamiento principal.
> Basicamente dice Arranca un contenedor Docker con MinIO, en segundo plano, expone los puertos 9000 y 9001, lo nombra minio, asegura que se reinicie solo si el servidor se apaga, guarda los datos en la carpeta /mnt/data del host, define el usuario y contraseña iniciales, y habilita la consola web en el puerto 9001 

#### 4\. Verificación de Estado

Confirmamos que el "banco" abrió sus puertas.

```bash
docker logs minio
```
> muestra los logs del contenedor llamado minio

> **Resultado Esperado:** Deberías ver un logo ASCII de MinIO y textos que dicen `API: http://...:9000` y `WebUI: http://...:9001`.
> **Nota de Seguridad:** Verás una advertencia en amarillo sobre la contraseña y el usuario por defecto. En producción esto es grave; en nuestra demo, es aceptable, pero menciónalo si te preguntan.

-----

### Prueba desde el navegador

Ahora vamos a probar si la Bóveda es accesible desde fuera.

1.  Abre el navegador en tu **Laptop Física**.
2.  Navega a `http://minio-vault:9001`, devido al dns que configuramos deberia funcionar o tambien puedes escribir la dirección IP de Tailscale de la VM1 (la que pusimos en `/etc/hosts` como `minio-vault`):
    `http://100.73.190.14:9001`.
3.  Deberías ver la pantalla de login de MinIO.
   ![Loggin de MinIO](Imagenes/MinIOLogin.png)
5.  Ingresa con `admin` y `SuperSecretKey123`.
   ![minio-vault_Dashboard1](Imagenes/minio-vault_Dashboard1.png)

Si ves el dashboard rojo/rosado de MinIO, **Fase 3 Completada al 90%**.
(Falta configurar el cliente `mc` para crear los buckets, que haremos en el siguiente paso).

¿Lograste entrar a la consola web? Di **"MinIO Operativo"**.


































-----

## Fase 4: Configuración de Servicios Críticos (Las Víctimas)
**Descripción:**
Puesta en marcha de los servicios que simulan la operación del negocio:
1.  **Servidor Web (App Node):** Nginx/Apache sirviendo una aplicación de demostración.
2.  **Base de Datos (DB Node):** MariaDB configurada para transacciones.
3.  **Acceso Público:** Configuración de *Port Forwarding* para permitir acceso desde dispositivos externos (móviles de la audiencia).

**Objetivo y Aporte:**
* Proveer los activos de información que serán protegidos.
* Demostrar la accesibilidad del servicio para el usuario final antes del "desastre".

-----

## Fase 5: Integridad de Datos y Volúmenes Lógicos (LVM)
**Descripción:**
Implementación de **LVM (Logical Volume Manager)** en el nodo de Base de Datos. Se migra el almacenamiento de MySQL/MariaDB a un volumen lógico dedicado, permitiendo la gestión avanzada del disco.

**Objetivo y Aporte:**
* **Atomicidad:** Habilitar la capacidad de tomar *Snapshots* (fotografías instantáneas) del disco.
* **Consistencia:** Asegurar que los backups de la base de datos se realicen sin corromper la información, incluso si el sistema está recibiendo escrituras en ese milisegundo.

-----

## Fase 6: Configuración del Motor de Resiliencia (Restic)
**Descripción:**
Instalación e inicialización de **Restic** en los nodos de aplicación y base de datos. Se configuran las variables de entorno para conectar con la Bóveda (MinIO) y se definen las claves de encriptación (AES-256).

**Objetivo y Aporte:**
* **Eficiencia:** Implementar la deduplicación de datos. Solo se almacenan los bloques que han cambiado, ahorrando espacio y ancho de banda.
* **Seguridad:** Garantizar que ningún dato salga del servidor sin estar cifrado (Encryption at Rest & in Transit).

-----

## Fase 7: Automatización y Planificación de Alta Frecuencia
**Descripción:**
Programación de **Systemd Timers** y Servicios (`.service` y `.timer`) para ejecutar los scripts de backup automáticamente. Se define la estrategia de retención (GFS) adaptada al evento (retención de minutos/horas).

**Objetivo y Aporte:**
* Eliminar el error humano en la ejecución de backups.
* Establecer un **RPO (Recovery Point Objective)** cercano a cero, realizando copias de seguridad cada minuto de forma transparente.
* Gestionar el ciclo de vida de los datos (borrado automático de backups obsoletos).

-----

## Fase 8: Orquestación de Recuperación (DRP como Código)
**Descripción:**
Desarrollo de Playbooks de **Ansible** en el nodo de Control. Estos scripts contienen la lógica para detener servicios, desmontar discos, descargar copias de seguridad desde la Bóveda y restaurar el sistema a un estado operativo.

**Objetivo y Aporte:**
* **Reducción del RTO (Recovery Time Objective):** Pasar de horas de restauración manual a segundos de restauración automática.
* **Idempotencia:** Asegurar que el proceso de recuperación sea repetible y libre de errores bajo presión.

-----

## Fase 9: Interfaz de Demostración y Control Visual
**Descripción:**
Creación de scripts interactivos (Menú de Mando) y monitores de estado en tiempo real. Esto permite al operador ejecutar ataques simulados y restauraciones quirúrgicas seleccionando IDs específicos de backups.

**Objetivo y Aporte:**
* Hacer visible lo invisible: Permitir que la audiencia "vea" los backups ocurriendo en tiempo real.
* Facilitar la operación durante la feria, abstrayendo la complejidad de los comandos de terminal en un menú intuitivo.

-----
