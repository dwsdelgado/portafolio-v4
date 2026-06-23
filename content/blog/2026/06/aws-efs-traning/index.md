---
date: '2026-06-12T09:00:00-05:00'
tags: ['aws', 'cloud', 'efs']
title: 'SAA-C03 — EFS: Elastic File System'
slug: 'Elastic-File-System'
---

{{< figure
  src="./Amazon-EFS.png"
  alt="Icono del servicio Amazon EFS"
  width="700"
  height="auto"
  class="insert-image"
>}}

Sistema de archivos compartido, **administrado por AWS** sobre protocolo **NFSv4.1** — el equivalente a un servidor NFS tradicional, pero sin que tú lo aprovisiones ni lo administres. Permite que varias EC2, contenedores (ECS/Fargate) o funciones Lambda monten el mismo filesystem al mismo tiempo.

<!--more-->

## EBS vs EFS vs S3

| | EBS | EFS | S3 |
|--|-----|-----|----|
| Tipo | Bloque | Archivo (NFS) | Objeto |
| Adjuntar a | 1 EC2 (o 16 con io2) | Múltiples EC2 simultáneo | No se monta — API/HTTP |
| AZ | 1 AZ | Multi-AZ | Global |
| Sistema de archivos | Tú lo formateas (xfs, ext4) | Ya viene formateado | No aplica |
| Escala | Manual (defines el tamaño) | Automática | Automática |
| OS | Linux y Windows | Solo Linux (para Windows usar FSx) | Cualquiera |
| Caso de uso | Disco de VM | Contenido compartido, CMS, home dirs | Backup, archivos estáticos, data lake |

---

## Clases de almacenamiento EFS

| Clase | Para qué | Costo |
|-------|----------|-------|
| **Standard** | Datos de acceso frecuente | Mayor |
| **Standard-IA** | Datos de acceso infrecuente (Infrequent Access) | ~92% más barato |
| **One Zone** | Una sola AZ, acceso frecuente | Más barato que Standard |
| **One Zone-IA** | Una sola AZ, acceso infrecuente | El más barato |

**EFS Lifecycle Management** — mueve archivos automáticamente entre clases según hace cuánto no se acceden, sin que tengas que mover nada a mano. Son 3 reglas independientes, cada una opcional:

| Regla | Qué hace | Se dispara por |
|---|---|---|
| **Transition into IA** | Mueve archivos de Standard → IA | Días sin acceso en Standard (7/14/30/60/90) |
| **Transition into Archive** | Mueve archivos a Archive — el tier más barato, para datos casi nunca tocados | Días sin acceso en Standard |
| **Transition into Standard** | Devuelve el archivo a Standard | El **primer acceso** (lectura o escritura) en IA o Archive — no es por tiempo |

**Por qué importa "Transition into Standard":** sin esta regla, un archivo que ya bajó a IA o Archive se queda ahí aunque lo empieces a usar seguido otra vez — pagando el costo/latencia de acceso de IA en cada request. Con la regla activa, EFS lo "promueve" de vuelta a Standard en cuanto detecta uso activo.

**Archive** es el tier más nuevo y más barato (más barato que IA) — pensado para datos de retención/compliance que casi nunca se leen. A cambio tiene mayor latencia en el primer byte que IA.

---

## Modos de rendimiento

| Modo | Cuándo usarlo |
|------|---------------|
| **General Purpose** (default) | La mayoría de casos — web servers, CMS, home dirs |
| **Max I/O** | Miles de EC2 accediendo simultáneo — Big Data, media processing |

---

## Modos de throughput

| Modo | Comportamiento |
|------|---------------|
| **Bursting** (default) | Throughput escala con el tamaño del filesystem |
| **Provisioned** | Defines el throughput independiente del tamaño |
| **Elastic** | Escala automáticamente según la carga — recomendado |

---

## Manos a la obra: EFS compartido entre dos EC2

{{< figure
  src="./efs-storage.png"
  alt="Diagrama de red de como se conectan las EC2 y EFS a traves del mount target"
  width="700"
  height="auto"
  class="insert-image"
>}}

### Creando el sistema de archivos

1. **EFS** → **Create file system** → Name: `training-efs` | VPC: `training-vpc`
2. **Customize**: Storage class `Standard`, Lifecycle según la tabla de arriba, Throughput `Elastic`, encriptación at-rest activada (sin costo extra con la key administrada por AWS)
3. **Network**: revisar que cree un Mount Target por AZ, con un Security Group que permita `NFS/2049`
4. **File system policy** (pantalla de "Policy options"):

   - Marcar solo **"Enforce in-transit encryption for all clients"** — fuerza TLS en el tráfico NFS, buena práctica real y tema de examen SAA. Obliga a montar con el **EFS mount helper** (`mount -t efs`) en vez de `mount -t nfs4` manual, porque el helper maneja el TLS automático vía `stunnel`
   - Dejar sin marcar "Prevent root access by default" y "Enforce read-only access by default" — romperían la prueba de escritura que hacemos más adelante en este lab
   - Dejar sin marcar "Prevent anonymous access" — no aplica aquí
5. **Create**

---

### Security Group para NFS

El mount target necesita recibir tráfico NFS (puerto 2049) desde las EC2:

1. EC2 → **Security Groups** → **Create security group**
2. Name: `training-sg-efs`
3. Inbound: `NFS` | Puerto `2049` | Source: el SG de tus EC2 (o el CIDR de la VPC)
4. **Create**
5. Volver a EFS → tu filesystem → **Network** → editar los mount targets → asignar `training-sg-efs`

{{< figure
  src="./sg-int.png"
  alt="Reglas inbound del security group training-sg-efs permitiendo NFS puerto 2049"
  width="700"
  height="auto"
  class="insert-image"
>}}

---

### Lanzando las instancias

EC2-1 en AZ-a, EC2-2 en AZ-b. Ambas con:
- IAM Role: `AmazonSSMManagedInstanceCore`
- SG: que permita salida NFS/2049 (o usar el mismo SG de las EC2)

---

### Montando el filesystem

En **ambas EC2** vía SSM. Como activamos **"Enforce in-transit encryption"** al crear el EFS,
el mount manual con `mount -t nfs4` (sin TLS) **va a fallar** — toca usar el **EFS mount
helper**, que maneja el TLS automáticamente vía `stunnel`:

```bash
# Instalar amazon-efs-utils
sudo dnf install -y amazon-efs-utils

# Crear punto de montaje
sudo mkdir /mnt/efs

# Montar (usa TLS automáticamente)
sudo mount -t efs fs-XXXXXXXX:/ /mnt/efs
```

---

### Verificando que el montaje es compartido

**En EC2-1:**
```bash
echo "escrito desde EC2-1" | sudo tee /mnt/efs/prueba.txt
```

{{< figure
  src="./ec2-1-efs-prueba.png"
  alt="Terminal de EC2-1 montando EFS y escribiendo el archivo de prueba"
  width="700"
  height="auto"
  class="insert-image"
>}}

> Fíjate en el `df -hT`: el filesystem aparece con `8.0E` (exabytes) de capacidad — no es un error, es justamente lo que significa "elástico": EFS no tiene un tamaño que tú definas, AWS reporta capacidad efectivamente ilimitada y solo pagas por lo que ocupas de verdad.

**En EC2-2:**
```bash
cat /mnt/efs/prueba.txt
# Resultado: escrito desde EC2-1
```

{{< figure
  src="./ec2-2-efs-prueba.png"
  alt="Terminal de EC2-2 leyendo el archivo escrito desde EC2-1, confirmando el filesystem compartido"
  width="700"
  height="auto"
  class="insert-image"
>}}

Ambas ven el mismo archivo en tiempo real — eso es EFS.

---

### Montaje persistente con fstab

Para que monte automáticamente al reiniciar:

```bash
# Agregar a /etc/fstab
echo "fs-XXXXXXXX:/ /mnt/efs efs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
```

---

## Conceptos clave:

| Pregunta | Respuesta |
|---|---|
| ¿Cuántas EC2 pueden montar EFS simultáneamente? | Miles — no hay límite definido |
| ¿EFS funciona en Windows? | No. Para Windows usar FSx for Windows File Server |
| ¿Cuál es la diferencia entre EFS y EBS? | EBS = disco de una EC2 (1 a 1). EFS = filesystem compartido entre muchas EC2 (N a N) |
| ¿EFS es Multi-AZ? | Sí — tiene mount targets en cada AZ, los datos se replican automáticamente |
| ¿Cómo se mueven datos a EFS Infrequent Access? | Con EFS Lifecycle Management — define los días sin acceso y AWS los mueve automáticamente |
| ¿Qué puerto usa NFS? | 2049. El Security Group del mount target debe permitir TCP/2049 desde las EC2 |
| ¿Pagas por el tamaño que defines o por lo que usas? | Por lo que usas — EFS es elástico, no defines capacidad |
