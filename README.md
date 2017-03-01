# Instalación de OpenNebula RDO


## Arquitectura del sistema

Para esta instalación básica usaremos 3 nodos físicos con el siguiente esquema y arquitectura de red:

<a data-flickr-embed="true" data-footer="true"  href="https://www.flickr.com/photos/manuparra/32868225721/in/dateposted-public/" title="OpenStack Basic Architecture"><img src="https://c1.staticflickr.com/4/3832/32868225721_62f7016426_z.jpg" width="640" height="453" alt="OpenStack Basic Architecture"></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

Nodos del diagrama:

- Control (**Controller**): nodecontroller 
- Computo (**Compute**): nodecompute -- Se encarga de la gestión/ejecución, etcétera de las MVs
- Red (**Network**): nodenetwork -- Exclusivamente dedicado al soporte de redes en la instalación de OpenStack. Hay muchas posibilidades, pero como recomendación, lo ideal es que sea un nodo dedicado a esta tarea y no se mezcle con los demás.

Algunas consideraciones sobre el diagrama:

- En nuestro caso todos los nodos tienen instalado **CentOS7 Minimal** como distribución linux
- Todos los nodos conectados a la misma red y switch por Ethernet (o Infiniband)
- Todos los nodos deben tener acceso a Internet
- Uno de los nodos será exclusivo e independiente para la gestión de redes (Neutron) de OpenStack.

Características de los nodos

- 32 cores.
- 256 GB RAM.
- 2 HD con 2TB cada uno.
- 4 tarjetas de red y 1 Infiniband, pero sólo se usará una conexión Ethernet.

## Datos de los nodos

- nodecontroller
```
Hostname 	: controller.dicits.es
IP Address 	:192.168.10.150
OS         	:CentOS 7
DNS  		:192.168.10.10
```

- nodecompute
```
Hostname 	: compute.dicits.es
IP Address 	:192.168.10.151
OS         	:CentOS 7
DNS  		:192.168.10.10
```

- nodenetwork
```
Hostname 	: network.dicits.es
IP Address 	:192.168.10.153
OS         	:CentOS 7
DNS  		:192.168.10.10
```

*En DNS, usamos el servidor de nombres de dominio que exista en la propia red.*

## Qué se instalará en cada nodo

- nodecontroller 
```
Keystone, Glance, swift, Cinder, Horizon, Neutron, Nova novncproxy, Novnc, Nova api, Nova Scheduler, Nova-conductor
```

- nodecompute
```
Nova Compute, Neutron – Openvswitch Agent
```

- nodenetwork
```
Neutron Server
Neturon DHCP agent
Neutron- Openswitch agent
Neutron L3 agent
```

## Instalación con RDO y PackStack

### Actualización de paquetes en CentOS 7

Una vez configurados los nodos con su IP correspondiente y validado el acceso a Internet de los mismos, pasamos a realizar un ``update`` para cada nodo: 

```
[root@nodecontroller]$ yum -y update ; reboot
```

```
[root@nodecompute]$ yum -y update ; reboot
```

```
[root@nodenetwork]$ yum -y update ; reboot
```

### Actualización de fichero hosts

Fijamos el nombre de cada uno de los nodos:

```
[root@nodenetwork]$ hostnamectl set-hostname nodecontroller
```

```
[root@nodenetwork]$ hostnamectl set-hostname nodecompute
```

```
[root@nodenetwork]$ hostnamectl set-hostname nodenetwork
```

Y añadimos la información correspondiente en ``/etc/hosts``:

```
192.168.10.150 nodecontroller.dicits.es nodecontroller
192.168.10.151 nodecompute.dicits.es    nodecompute
192.168.10.152 nodenetwork.dicits.es    nodenetwork
```

### Deshabilitar SELINUX y NetworkManager

En cada uno de los nodos modificar el fichero ``/etc/sysconfig/selinux`` cambiar ``SELINUX=disabled`` 

También deshabilitar NetworkManager en cada uno de los nodos:

```
systemctl stop NetworkManager
systemctl disable NetworkManager
```

Una vez hechos los cambios reinicar de nuevo los tres nodos:

```
reboot
```

### Configurar SSH passwordless desde el nodo de control a los demás.

Esto es importante, ya que permite que, entre los nodos, se pueda conectar con SSH sin utilizar la clave. En este caso lo que se usa es la clave pública, para autenticar dentro del grupo de nodos que tenemos:

En el ``nodecontroller``:

```
[root@controller ]# ssh-keygen
[root@controller ]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.151
[root@controller ]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.152
```

Hecho esto, chequea que puedes conectarte con SSH a cada uno de los nodos sin que pida la clave por teclado:

```
[root@nodecontroller ~]# ssh root@192.168.10.151
[root@nodecompute ~]# 
```

Si has conectado correctamente, este paso está bien realizado.






## Añadiendo espacio a CINDER

Comprobamos en el nodo de control (``controller``) el espacio que hay disponible. Por defecto en la instalación de RDO, sólo se instalan 10GB de espacio y esto **NO** es suficiente, así que tendremos que ampliarlo.

```
[root@controller ]# vgs
  VG             #PV #LV #SN Attr   VSize  VFree 
  cinder-volumes   1   1   0 wz--n- 20.60g 10.60g
```

Ahora añadimos un nuevo volumen para CINDER:

```
[root@controller ~]# dd if=/dev/zero of=cinder-volumes bs=1 count=0 seek=300G
[root@controller ~]# losetup /dev/loop3 cinder-volumes
[root@controller ~]# fdisk /dev/loop3
```

Usar la secuencia de teclas siguiente para el menú de ``fdisk``:

```
n
p
1
ENTER
ENTER
t
8e
w
```

Creamos un dispositivo físico:

```
[root@controller ~]# pvcreate /dev/loop3
```

Extendemos el volumen de cinder añadiendo el nuevo creado:

```
[root@controller ~]# vgextend cinder-volumes  /dev/loop3
```

Comprobamos que se ha añadido el espacio a CINDER:

```
[root@controller ~]# vgs
  VG             #PV #LV #SN Attr   VSize   VFree  
  cinder-volumes   2   1   0 wz--n- 320.59g 310.59g
```

Si no has configurado en ``packstack`` cuales son los servidores de almacenamiento o dónde está CINDER y el tamaño, puedes hacerlo en la instalación indicando qué servidores estarán disponibles para usar espacio de almacenamiento para CINDER.



# Afinar la instalación de OpenNebula

## Añadir más nodos de almacenamiento con CINDER


## Añadir más nodos de RED (network)



# Referencias



