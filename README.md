<img src="http://zdnet2.cbsistatic.com/hub/i/r/2016/10/25/e2569fbb-c7f8-4f5c-9b81-f841816e5261/resize/770xauto/b291ee9185e8472b75b6875fda72b3f4/openstack-logo-2016.png" width="600">


# Instalación y despliegue de OpenStack

Manual completo y funcional de instalación, despliegue y uso de OpenStack en modo MULTI-NODO sobre CentOS7 con RDO para la versión de OpenStack **NEWTON**. Este manual se ha desarrollado para poner en producción un cluster completo con todos los servicios de OpenStack, desde la instalación básica hasta aspectos más complejos de despliegue, uso y automatización.

En este tutorial cubre los siguientes aspectos:

- Instalación de OpenStack sobre tres nodos: **Control, Computo y Red**.
- Instalación adicional de nodos (+ de 3) para diversos roles control, computo y red.
- Alta disponibilidad en OpenStack.
- Gestión de OpenStack desde línea de comandos [en proceso].
- Despligue de servicios de BigData sobre OpenStack: HDFS, Hadoop, Spark, etc. [pendiente].

## Autoría

Este manual completo está desarrollado por Rubén Castro, Juan A. Cortés y Manuel J. Parra dentro del marco de trabajo del Laboratorio  Distributed Computational Intelligence and Time Series y el departamento de Ciencias de la Computación e Inteligencia Artificial de la Universidad de Granada, para el proyecto Minería de Datos en CloudComputing.


### Redistribución del tutorial

Recuerda citar a los autores y la referencia a este repositorio si usas el material que aquí se expone.


Contenido
=================

   * [Instalación de OpenStack RDO](#instalación-de-openstack-rdo)
      * [Arquitectura del sistema](#arquitectura-del-sistema)
      * [Datos de los nodos](#datos-de-los-nodos)
      * [Qué se instalará en cada nodo](#qué-se-instalará-en-cada-nodo)
      * [Instalación con RDO y PackStack](#instalación-con-rdo-y-packstack)
         * [Actualización de paquetes en CentOS 7](#actualización-de-paquetes-en-centos-7)
         * [Actualización de fichero hosts](#actualización-de-fichero-hosts)
         * [Deshabilitar SELINUX y NetworkManager](#deshabilitar-selinux-y-networkmanager)
         * [Configurar SSH passwordless desde el nodo de control a los demás.](#configurar-ssh-passwordless-desde-el-nodo-de-control-a-los-demás)
         * [Habilitar el repositorio de RDO e instalar PackStack](#habilitar-el-repositorio-de-rdo-e-instalar-packstack)
         * [Generar y personalizar el fichero de instalación](#generar-y-personalizar-el-fichero-de-instalación)
         * [Instalar OpenStack](#instalar-openstack)
      * [Añadiendo espacio a CINDER](#añadiendo-espacio-a-cinder)
   * [Afinar la instalación de OpenStack](#afinar-la-instalación-de-openstack)
      * [Añadir más nodos de almacenamiento con CINDER](#añadir-más-nodos-de-almacenamiento-con-cinder)
      * [Añadir más nodos de RED (network)](#añadir-más-nodos-de-red-network)
	  * [Habilitar Sahara DataProcessing en Horizon](#horizonabilitar-dataprocessing-con-sahara-en-horizon)
   * [ Referencias](#referencias)


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
- Las particiones de disco son particiones estandar de 900 GB para la partición raíz / y el resto para swap (no LVM)
- Uno de los nodos será exclusivo e independiente para la gestión de redes (Neutron) de OpenStack.

Características de los nodos

- 32 cores.
- 256 GB RAM.
- 2 HD con 2TB cada uno.
- 4 tarjetas de red y 1 Infiniband, pero sólo se usará una de ellas.

La **instalación mínima** admitible tiene que tener la siguiente configuración de los nodos:

- 8 cores
- 8 GB de RAM
- 1 HD con 1 TB
- 1 Tarjeta de red para cada nodo

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

*En DNS, usamos el servidor de nombres de dominio que exista en la propia red. En nuestro caso ``192.168.10.10`` *

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

NOTA: Este tutorial esta orientado a la instalación de packstack 9.0.1. (que corresponde a OpenStack NEWTON). Hay que tener en cuenta que si se instala packstack por defecto tomará la última versión disponible del repositorio (Actualmente la 10.0.0 que corresponde con OpenStack OCATA).


### Actualización de fichero hosts

Una vez configurados los nodos con su IP correspondiente y validado el acceso a Internet de los mismos, fijamos el nombre de cada uno de los nodos (``hostname``):

Para el nodo controlador (IP: 192.168.6.150), ponemos el nombre:
```
hostnamectl set-hostname nodecontroller
```

Para el nodo de computo (IP: 192.168.6.151), ponemos el nombre:
```
hostnamectl set-hostname nodecompute
```

Para el nodo de red (IP: 192.168.6.152), ponemos el nombre:
```
hostnamectl set-hostname nodenetwork
```

Y añadimos el siguiente esquema en ``/etc/hosts`` en cada uno de los nodos :

```
192.168.10.150 nodecontroller.dicits.es nodecontroller
192.168.10.151 nodecompute.dicits.es    nodecompute
192.168.10.152 nodenetwork.dicits.es    nodenetwork
```

*En lugar de ``.dicits.es`` puedes poner algo como ``miorganizacion.com`` o similar que corresponda a tu organización*


### Actualización de paquetes en CentOS 7

Pasamos a realizar un ``update`` para cada nodo:

```
[root@nodecontroller]$ yum -y update ; reboot
```

```
[root@nodecompute]$ yum -y update ; reboot
```

```
[root@nodenetwork]$ yum -y update ; reboot
```


### Deshabilitar SELINUX y NetworkManager

En cada uno de los nodos modificar el fichero ``/etc/sysconfig/selinux`` cambiar ``SELINUX=disabled``

También deshabilitar NetworkManager en cada uno de los nodos:

```
systemctl stop NetworkManager
systemctl disable NetworkManager
```

Una vez hechos los cambios es necesario reinicar de nuevo los tres nodos:

```
reboot
```

### Configurar SSH passwordless desde el nodo de control a los demás.

Esto es importante, ya que permite que, entre los nodos, se pueda conectar con SSH sin utilizar la clave. En este caso lo que se usa es la clave pública, para autenticar dentro del grupo de nodos que tenemos (y además es requerido por packstack para poder hacer la instalación):

En el ``nodecontroller`` (cuando el comando ``ssh-keygen`` pida clave, dejala en blanco):

```
[root@nodecontroller ]# ssh-keygen
[root@nodecontroller ]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.151
[root@nodecontroller ]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.152
```


Hecho esto, chequea que puedes conectarte con SSH a cada uno de los nodos sin que pida la clave por teclado:

```
[root@nodecontroller ~]# ssh root@192.168.10.151
[root@nodecompute ~]#
```

Si has conectado correctamente, este paso está bien realizado. En caso contrario vuelve a repetirlo.

### Habilitar el repositorio de RDO e instalar PackStack

Con los siguientes comandos añadiremos a nuestro sistema el paquete RPM de RDO, donde se encuentra PackStack 9.0.1 (OpenStack NEWTON), y lo instalaremos:

Serán ejecutados en ``nodecontroller`` (esto instalará la versión de OpenStack Newton [9.0.1] ):

```
[root@nodecontroller ]# yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-newton/rdo-release-newton-4.noarch.rpm
[root@nodecontroller ]# yum install -y openstack-packstack
```
### Generar y personalizar el fichero de instalación

Con el siguiente comando generaremos el fichero de instalación ("Answer File" según PackStack).

En el ``nodecontroller``:

```
[root@nodecontroller ]# packstack --gen-answer-file=/root/instalacion.txt
```

Editaremos este fichero modificando las opciones que nos interesen. Para esta instalación añadiremos las IP de nuestros nodos, desactivaremos la demostración, desactivaremos Ceilometer, añadiremos un servidor para NTP y cambiaremos las contraseñas de los servicios que nos interesen.


En el ``nodecontroller`` abrir el fichero ``/root/instalacion.txt`` y buscar y cambiar las siguientes directivas:
```
[root@controller ~]# vi /root/instalacion.txt
........................................
CONFIG_CONTROLLER_HOST=192.168.10.150
CONFIG_COMPUTE_HOSTS=192.168.10.151
CONFIG_NETWORK_HOSTS=192.168.10.153
CONFIG_PROVISION_DEMO=n
CONFIG_CEILOMETER_INSTALL=n
CONFIG_HORIZON_SSL=y
CONFIG_NTP_SERVERS= 1.es.pool.ntp.org
CONFIG_KEYSTONE_ADMIN_PW= <contraseña>
..........................................
```
El servidor NTP deberá ser el propio de nuestra región, por ejemplo ``hora.ugr.es``.

### Instalar OpenStack

Una vez editado el fichero de configuración ```/root/instalacion.txt``` pasamos
a realizar la instalación de OpenStack con PackStack. Para ello lanzamos el siguiente comando:
En el ``nodecontroller``:

```
[root@controller ~]# packstack --answer-file=/root/instalacion.txt
```

En la figura vemos cómo ha finalizado correctamente la instalación.
![install_001](https://github.com/manuparra/openstack/blob/master/imgs/img_001.png?raw=true)

Una vez realizada la instalación sin ningún error podremos ver como se ha
creado una nueva interfaz de red en el nodo de red (``nodenetwork``), llamada ``br-ex``. Ésta se puede ver ejecutando ``ifconfig -a`` como muestra la imagen.

![install_002](https://github.com/manuparra/openstack/blob/master/imgs/img_002.png?raw=true)

Una vez comprobado
que realmente está la interfaz de red en el nodo de red, hacemos que esta nueva
interfaz sea la que de acceso al exterior mientras que la otra interconectará los nodos. Para ello copiamos la configuración de la interfaz existente a la nueva interfaz ```br-ex``` y la editamos.
En el ``nodenetwork``:
```
[root@network ~]# cd /etc/sysconfig/network-scripts/
[root@network network-scripts]# cp ifcfg-enp0s3 ifcfg-br-ex
[root@network network-scripts]# vi ifcfg-enp0s3
........................................
DEVICE=enp1s0
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
ONBOOT=yes
........................................

........................................
[root@network network-scripts]# vi ifcfg-br-ex
DEVICE=br-ex
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
IPADDR=192.168.10.153
NETMASK=255.255.255.0
GATEWAY=192.168.10.1
DNS1=192.168.10.10
DNS2=8.8.8.8
ONBOOT=yes
........................................
```

Acabada la edición, pasamos a reiniciar el servicio de red.
En el `nodenetwork`:
```
[root@network ~]# systemctl restart network
```
Comprobamos que todos los cambios han surtido efecto ejecutando de nuevo ``ifconfig``.
![install_003](https://github.com/manuparra/openstack/blob/master/imgs/img_003.png?raw=true)


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

Si no has configurado en ``packstack`` cuáles son los servidores de almacenamiento o dónde está CINDER y el tamaño, puedes hacerlo en la instalación indicando qué servidores estarán disponibles para usar espacio de almacenamiento para CINDER.



# Afinar la instalación de OpenStack

## Añadir más nodos de almacenamiento con CINDER


## Añadir más nodos de RED (network)


## Habilitar DataProcessing con Sahara en Horizon

Primero hay que instalar el paquete siguiente:

```
pip install sahara-dashboard
```

Una vez hecho esto, cambiar la configuración de `Horizon`:

```
vi /usr/share/openstack-dashboard/openstack_dashboard/settings.py
```

y añadir a la sección ``INSTALLED_APPS`` el valor `saharadashboard`: 

```
INSTALLED_APPS = [
    'openstack_dashboard',
    'django.contrib.contenttypes',
    'django.contrib.auth',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django.contrib.humanize',
    'django_pyscss',
    'openstack_dashboard.django_pyscss_fix',
    'compressor',
    'horizon',
    'openstack_auth',
    'saharadashboard',
]
```

Finalmente reiniciar apache:

```
service httpd restart
```
## Personalizar el arranque con cloud-init.

En el caso de que tengamos que realizar ciertas tareas durante el arranque de nuestras imágenes, podemos recurrir a *cloud-init*. Así pues, durante la configuración de nuestra nueva instancia podemos añadir nuestro *script* en el siguiente menú:

![cloudinit](https://i.imgur.com/S1J44vt.png)

Esto es muy útil ya que algunas imágenes de ciertos sistemas operativos (Ubuntu, por ejemplo) requieren la inyección de una clave SSH para conectar y de esta manera podemos inyectarla al usuario que queramos (entre otras cosas). Por ejemplo, para crear un usuario, añadirle una clave y modificar los DNS podríamos usar el siguiente *script*:

```
#cloud-config
users:
  - name: tfg_user
    ssh-authorized-keys:
      - ssh-rsa <clave generado por OpenStack>
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups: sudo
    shell: /bin/bash
runcmd:
  - echo 'interface "ens3" {prepend domain-name-servers 8.8.8.8;}' | sudo tee --append /etc/dhcp/dhclient.conf
  - sudo reboot
```

Para ver otras posibles opciones, se enlazan los ejemplos de la página oficial: [Cloud config examples](https://cloudinit.readthedocs.io/en/latest/topics/examples.html)

# Referencias
