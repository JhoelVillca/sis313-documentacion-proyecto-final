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

## 🏗️ Arquitectura de la Solución

El sistema se distribuye en **4 Nodos Lógicos** interconectados, diseñados para simular un entorno de producción real donde los servicios (App/DB), el almacenamiento (Backups) y la gestión (Control) están desacoplados para garantizar la supervivencia de los datos incluso si los servidores principales son comprometidos.



[Image of network star topology]


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
## 📅 Estrategia de Retención de Datos

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

> **Resiliencia no es solo guardar datos, es saber qué versión de la verdad recuperar.**

***

# Fases de Implementación

Para garantizar la replicabilidad y el éxito del **Sistema de Resiliencia Operativa**, hemos dividido la ejecución en 9 fases estratégicas. Cada fase construye una capa de funcionalidad sobre la anterior, desde la infraestructura física hasta la interfaz de usuario final.

-----

## 📍 Fase 1: Aprovisionamiento de Infraestructura Base
**Descripción:**
Consiste en la creación y configuración inicial de los 4 Nodos Virtuales (VMs) que compondrán el sistema. Se establecen los recursos de hardware virtual (CPU, RAM, Disco) y se instala el sistema operativo base (Ubuntu Server).

**Objetivo y Aporte:**
* Establecer los cimientos del sistema.
* Segregar funciones: Separar la lógica de negocio (App/DB), el almacenamiento (Bóveda) y la gestión (Control) para evitar un "Punto Único de Fallo" (SPOF).

-----

## 📍 Fase 2: Implementación de Red de Malla (Overlay Network)
**Descripción:**
Despliegue de una red privada virtual (VPN de malla) utilizando **Tailscale/ZeroTier**. Esto crea una capa de red abstracta sobre la infraestructura física, permitiendo que las máquinas se comuniquen de forma segura y encriptada sin depender de la configuración del router local (Wi-Fi de la feria o laboratorio).

**Objetivo y Aporte:**
* **Portabilidad Total:** El sistema funciona idénticamente en el laboratorio, en una feria pública o en Internet.
* **Independencia de IP:** Uso de *MagicDNS* para resolver nombres (`db-node`, `app-node`) en lugar de depender de IPs estáticas frágiles.
* **Seguridad:** Todo el tráfico entre nodos viaja cifrado, inmune a espionaje en redes públicas.

-----

## 📍 Fase 3: Despliegue de la Bóveda Inmutable (Object Storage)
**Descripción:**
Instalación y configuración de **MinIO** (compatible con Amazon S3) en un entorno contenerizado (Docker). Se configuran las políticas de acceso, usuarios de servicio y persistencia de datos en disco.

**Objetivo y Aporte:**
* Crear un repositorio centralizado y desacoplado para los backups.
* Simular una arquitectura de nube real (Cloud-Native) en un entorno local.
* Garantizar que si los servidores de aplicación son destruidos, los datos de respaldo permanezcan intactos en un "búnker" aislado.

-----

## 📍 Fase 4: Configuración de Servicios Críticos (Las Víctimas)
**Descripción:**
Puesta en marcha de los servicios que simulan la operación del negocio:
1.  **Servidor Web (App Node):** Nginx/Apache sirviendo una aplicación de demostración.
2.  **Base de Datos (DB Node):** MariaDB configurada para transacciones.
3.  **Acceso Público:** Configuración de *Port Forwarding* para permitir acceso desde dispositivos externos (móviles de la audiencia).

**Objetivo y Aporte:**
* Proveer los activos de información que serán protegidos.
* Demostrar la accesibilidad del servicio para el usuario final antes del "desastre".

-----

## 📍 Fase 5: Integridad de Datos y Volúmenes Lógicos (LVM)
**Descripción:**
Implementación de **LVM (Logical Volume Manager)** en el nodo de Base de Datos. Se migra el almacenamiento de MySQL/MariaDB a un volumen lógico dedicado, permitiendo la gestión avanzada del disco.

**Objetivo y Aporte:**
* **Atomicidad:** Habilitar la capacidad de tomar *Snapshots* (fotografías instantáneas) del disco.
* **Consistencia:** Asegurar que los backups de la base de datos se realicen sin corromper la información, incluso si el sistema está recibiendo escrituras en ese milisegundo.

-----

## 📍 Fase 6: Configuración del Motor de Resiliencia (Restic)
**Descripción:**
Instalación e inicialización de **Restic** en los nodos de aplicación y base de datos. Se configuran las variables de entorno para conectar con la Bóveda (MinIO) y se definen las claves de encriptación (AES-256).

**Objetivo y Aporte:**
* **Eficiencia:** Implementar la deduplicación de datos. Solo se almacenan los bloques que han cambiado, ahorrando espacio y ancho de banda.
* **Seguridad:** Garantizar que ningún dato salga del servidor sin estar cifrado (Encryption at Rest & in Transit).

-----

## 📍 Fase 7: Automatización y Planificación de Alta Frecuencia
**Descripción:**
Programación de **Systemd Timers** y Servicios (`.service` y `.timer`) para ejecutar los scripts de backup automáticamente. Se define la estrategia de retención (GFS) adaptada al evento (retención de minutos/horas).

**Objetivo y Aporte:**
* Eliminar el error humano en la ejecución de backups.
* Establecer un **RPO (Recovery Point Objective)** cercano a cero, realizando copias de seguridad cada minuto de forma transparente.
* Gestionar el ciclo de vida de los datos (borrado automático de backups obsoletos).

-----

## 📍 Fase 8: Orquestación de Recuperación (DRP como Código)
**Descripción:**
Desarrollo de Playbooks de **Ansible** en el nodo de Control. Estos scripts contienen la lógica para detener servicios, desmontar discos, descargar copias de seguridad desde la Bóveda y restaurar el sistema a un estado operativo.

**Objetivo y Aporte:**
* **Reducción del RTO (Recovery Time Objective):** Pasar de horas de restauración manual a segundos de restauración automática.
* **Idempotencia:** Asegurar que el proceso de recuperación sea repetible y libre de errores bajo presión.

-----

## 📍 Fase 9: Interfaz de Demostración y Control Visual
**Descripción:**
Creación de scripts interactivos (Menú de Mando) y monitores de estado en tiempo real. Esto permite al operador ejecutar ataques simulados y restauraciones quirúrgicas seleccionando IDs específicos de backups.

**Objetivo y Aporte:**
* Hacer visible lo invisible: Permitir que la audiencia "vea" los backups ocurriendo en tiempo real.
* Facilitar la operación durante la feria, abstrayendo la complejidad de los comandos de terminal en un menú intuitivo.

-----
