# 🚀 Proyecto Final SIS313: [Título del Proyecto]

> **Asignatura:** SIS313: Infraestructura, Plataformas Tecnológicas y Redes<br>
> **Semestre:** 2/2025<br>
> **Docente:** Ing. Marcelo Quispe Ortega

## 👥 Miembros del Equipo G-11

| Nombre Completo | Rol en el Proyecto | Contacto (GitHub/Email) |
| :--- | :--- | :--- |
| Pomacahua Cardoso Benjamin | Db Node | [Benjamin](https://github.com/BPC-369) |
| Fernando Jose Quispe Gardeazabal | DRP Control  | [FernandoQuispe](https://github.com/FerchoJQG)  |
| Jhoel Mauricio Villca Villca | Boveda  | [JhoelVillca](https://github.com/JhoelVillca) |
| Alan Jesus Uzeda Rivera | APP Node | ----- |

## 🎯 I. Objetivo del Proyecto

Describe el objetivo de manera puntual, debe ser específica y medible, tal como se define en el banco de proyectos o tal como lo plantean como proyecto.

> **Objetivo:** Diseñar e implementar un sistema de Backups Automáticos que utilice una estrategia incremental eficiente, gestione la retención y transforme el Plan de Recuperación ante Desastres (DRP) en código ejecutable para garantizar la integridad transaccional y minimizar el RTO.

## 💡 II. Justificación e Importancia

Explica por qué este proyecto es relevante para una infraestructura universitaria o empresarial. Menciona los problemas de la continuidad operacional (T1) o la seguridad (T5) que resuelve.

> **Justificación:** Los métodos tradicionales de respaldo (scripts de copia simples) fallan al garantizar la integridad de bases de datos en caliente y dependen de procesos manuales lentos durante una crisis. Este proyecto es vital para la Continuidad Operacional (T1), ya que desacopla el almacenamiento de la infraestructura de cómputo, elimina el error humano mediante la automatización (T6) y asegura la Seguridad (T5) aplicando cifrado en reposo y en tránsito, garantizando que el negocio sobreviva incluso a la destrucción total de sus servidores principales.

## 🛠️ III. Tecnologías y Conceptos Implementados

### 3.1. Tecnologías Clave

Enumera y describe brevemente el rol de cada software y tecnología utilizada.

* **Restic:** Función específica: Motor de backup con deduplicación de datos y cifrado nativo AES-256.
* **MinIO:** Función específica: Almacenamiento de Objetos (S3 Compatible) que actúa como "Bóveda" inmutable y aislada.
* **LVM (Logical Volume Manager):** Función específica: Gestión de snapshots para "congelar" el disco y garantizar consistencia atómica en la BD.
* **Ansible:** Función específica: Orquestación del DRP (Infrastructure as Code) para la restauración automatizada de servicios.
* **Tailscale:** Función específica: Red Overlay (Mesh VPN) para garantizar conectividad segura entre nodos independientemente de la red física.
* **Systemd Timers:** Función específica: Planificación de alta frecuencia y gestión de logs de los servicios de respaldo.

### 3.2. Conceptos de la Asignatura Puestos en Práctica (T1 - T6)

Marca con un ✅ los temas avanzados de la asignatura que fueron implementados:

* **Alta Disponibilidad (T2) y Tolerancia a Fallos:** ✅ Desacoplamiento del almacenamiento (MinIO) para supervivencia de datos ante desastres en nodos de aplicación.
* **Seguridad y Hardening (T5):** ✅ Implementación de "Encryption at Rest" (Restic), túneles cifrados (Tailscale) y Hardening SSH mediante llaves Ed25519.
* **Automatización y Gestión (T6):** ✅ Implementación de DRP como Código (Ansible Playbooks) y automatización de backups con Systemd.
* **Balanceo de Carga/Proxy (T3/T4):**  *(No aplica en esta arquitectura enfocada en DRP)*
* **Monitoreo (T4/T1):** ✅ Implementación de Dashboard en tiempo real (`monitor.sh`) para observabilidad de la creación de snapshots.
* **Networking Avanzado (T3):** ✅ Implementación de Red Overlay para abstracción de infraestructura física y Port Forwarding (NAT) para acceso público.


## 🌐 IV. Diseño de la Infraestructura y Topología

### 4.1. Diseño Esquemático

Incluye un diagrama de la topología final. Muestra claramente la segmentación de red, las IPs utilizadas, y los flujos de tráfico.

> 
| VM/Host | Rol | IP Overlay (Tailscale) | Red Lógica | SO |
| :--- | :--- | :--- | :--- | :--- |
| **VM1 (minio-vault)** | Bóveda de Almacenamiento (S3) | 100.x.y.z | Red Mesh | Ubuntu 22.04 |
| **VM2 (app-node)** | Servidor Web (Víctima 1) | 100.x.y.z | Red Mesh | Ubuntu 22.04 |
| **VM3 (db-node)** | Base de Datos + LVM (Víctima 2) | 100.x.y.z | Red Mesh | Ubuntu 22.04 |
| **VM4 (drp-control)**| Cerebro / Control Ansible | 100.x.y.z | Red Mesh | Ubuntu 22.04 |


### 4.2. Estrategia Adoptada (Opcional)

Describe la estrategia de diseño y las decisiones críticas.

* **Estrategia "Snapshot-First" (Integridad):** Se priorizó la consistencia de datos sobre la velocidad de copia pura. Antes de cada backup de BD, se utiliza LVM para crear una "instantánea" del sistema de archivos, garantizando que no se copien transacciones a medias.
* **Estrategia de Recuperación Quirúrgica:** Se implementó un menú interactivo ("Time Travel") que permite seleccionar versiones específicas de los backups en lugar de restaurar ciegamente la última versión, protegiendo contra errores lógicos recientes.
## 📋 V. Guía de Implementación y Puesta en Marcha

Documenta los pasos esenciales para que cualquier persona pueda replicar el proyecto (instalación, configuración de ficheros clave, comandos).

### 5.1. Pre-requisitos
  * 4 Máquinas Virtuales (o físicas) con Ubuntu Server 22.04/24.04.
  * Acceso root/sudo y conectividad a Internet para instalación de paquetes.
  * Discos secundarios virtuales configurados en VM1 (20GB) y VM3 (10GB).
### 5.2. Despliegue (Ejecución de la Automatización)
1.  **Red:** Instalar Tailscale en todos los nodos y configurar `/etc/hosts` como redundancia DNS.
2.  **Almacenamiento:** Formatear discos secundarios, montar `/mnt/data` y desplegar contenedor MinIO en VM1.
3.  **Servicios:** Desplegar LAMP Stack en VM2/VM3 y configurar LVM en VM3 (`vg_datos/lv_mysql`).
4.  **Resiliencia:** Inicializar repositorios Restic en VM2/VM3 apuntando a VM1 y activar Systemd Timers.
5.  **Control:** Configurar inventario de Ansible en VM4, intercambiar llaves SSH y desplegar scripts de menú.

### 5.3. Ficheros de Configuración Clave
  * `/usr/local/bin/backup_db.sh`: Script crítico que orquesta el congelamiento LVM y la ejecución de Restic.
  * `/home/admin-drp/ansible-drp/restore_db.yml`: Playbook de Ansible para la restauración automatizada, limpieza y trasplante de datos.
  * `/home/admin-drp/demo/menu.sh`: Interfaz de Centro de Mando para gestión de crisis y selección de snapshots.
  * `/etc/systemd/system/backup-db.timer`: Planificador de alta frecuencia (minuto a minuto).

**Incluir además los archivos de configuración y software a utilizar dentro del proyecto y organizados en carpetas.**

## ⚠️ VI. Pruebas y Validación

| Prueba Realizada | Resultado Esperado | Resultado Obtenido |
| :--- | :--- | :--- |
| **Simulación de Ataque Web** (Borrado de `index.php`) | El sitio debe devolver Error 404 y recuperarse automáticamente tras ejecutar Ansible. | **[ÉXITO]** Recuperado en \< 10s. |
| **Destrucción de Base de Datos** (`rm -rf /var/lib/mysql`) | El servicio MariaDB debe fallar. Tras la restauración, los datos transaccionales deben reaparecer intactos. | **[ÉXITO]** Datos íntegros verificados. |
| **Integridad de Snapshot LVM** | El backup no debe bloquear la base de datos ni corromper archivos abiertos durante la escritura. | **[ÉXITO]** Backup realizado en caliente sin errores. |


## 📚 VII. Conclusiones y Lecciones Aprendidas

Se logró implementar una arquitectura resiliente capaz de recuperar servicios críticos en segundos, cumpliendo el objetivo de automatización (T6) y continuidad (T1).
Es importante no solo enfocarse en evitar que una web se caiga, si no este proyecto nos ilumino haciendo darnos cuenta que tambien hay que pensar en que pasa si la web se cae.
