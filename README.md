<img src="http://zdnet2.cbsistatic.com/hub/i/r/2016/10/25/e2569fbb-c7f8-4f5c-9b81-f841816e5261/resize/770xauto/b291ee9185e8472b75b6875fda72b3f4/openstack-logo-2016.png" width="600">


# Instalación y despliegue de OpenStack

Manual completo y funcional de instalación, despliegue y uso de OpenStack en modo MULTI-NODO sobre CentOS7 con RDO. Este manual se ha desarrollado para poner en producción un cluster completo con todos los servicios de OpenStack, desde la instalación básica hasta aspectos más complejos de despliegue, uso y automatización.

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
         * [Acceder a la interfaz web](#acceder-a-la-interfaz-web)
         * [Crear proyectos y añadir usuarios](#crear-proyectos-y-añadir-usuarios)
            * [Crear un "sabor" e imagen](#crear-un-sabor-e-imagen)
            * [Crear la red y el router del proyecto](#crear-la-red-y-el-router-del-proyecto)
            * [Añadir grupos de seguridad y claves](#añadir-grupos-de-seguridad-y-claves)
         * [Lanzar una instancia](#lanzar-una-instancia)
      * [Añadiendo espacio a CINDER](#añadiendo-espacio-a-cinder)
   * [Afinar la instalación de OpenStack](#afinar-la-instalación-de-openstack)
      * [Añadir más nodos de almacenamiento con CINDER](#añadir-más-nodos-de-almacenamiento-con-cinder)
      * [Añadir más nodos de RED (network)](#añadir-más-nodos-de-red-network)
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

La instalación mínima es admitible es:

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



### Actualización de fichero hosts

Una vez configurados los nodos con su IP correspondiente y validado el acceso a Internet de los mismos, fijamos el nombre de cada uno de los nodos (``hostname``):

Para el nodo controlador (IP: 192.168.6.50), ponemos el nombre:
```
hostnamectl set-hostname nodecontroller
```

Para el nodo de computo (IP: 192.168.6.51), ponemos el nombre:
```
hostnamectl set-hostname nodecompute
```

Para el nodo de red (IP: 192.168.6.52), ponemos el nombre:
```
hostnamectl set-hostname nodenetwork
```

Y añadimos el siguiente esquema en ``/etc/hosts`` en cada uno de los nodos :

```
192.168.10.150 nodecontroller.dicits.es nodecontroller
192.168.10.151 nodecompute.dicits.es    nodecompute
192.168.10.152 nodenetwork.dicits.es    nodenetwork
```

*En lugar de ``.dicits.es`` puedes poner algo como ``miorganizacion.com`` o similar*


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

Una vez hechos los cambios reinicar de nuevo los tres nodos:

```
reboot
```

### Configurar SSH passwordless desde el nodo de control a los demás.

Esto es importante, ya que permite que, entre los nodos, se pueda conectar con SSH sin utilizar la clave. En este caso lo que se usa es la clave pública, para autenticar dentro del grupo de nodos que tenemos:

En el ``nodecontroller``:

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

Si has conectado correctamente, este paso está bien realizado.

### Habilitar el repositorio de RDO e instalar PackStack

Con los siguientes comandos añadiremos a nuestro sistema el repositorio de RDO, donde se encuentra PackStack, y lo instalaremos:

Serán ejecutados en ``nodecontroller``:
```
[root@nodecontroller ]# yum install -y https://www.rdoproject.org/repos/rdo-release.rpm
[root@nodecontroller ]# yum install -y openstack-packstack
```
### Generar y personalizar el fichero de instalación

Con el siguiente comando generaremos el fichero de instalación ("Answer File" según PackStack).

En el ``nodecontroller``:
```
[root@nodecontroller ]# packstack --gen-answer-file=/root/instalacion.txt
```
Editaremos este fichero modificando las opciones que nos interesen. Para esta instalación añadiremos las IP de nuestros nodos, desactivaremos la demostración, desactivaremos Ceilometer, añadiremos un servidor para NTP y cambiaremos las contraseñas de los servicios que nos interesen.
En el ``nodecontroller``:
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
El servidor NTP deberá ser el propio de nuestra región.

### Instalar OpenStack

Una vez editado el fichero de configuración ```/root/instalacion.txt``` pasamos
a realizar la instalación de OpenStack con PackStack. Para ello lanzamos el siguiente comando:
En el ``nodecontroller``:
```
[root@controller ~]# packstack --answer-file=/root/instalacion.txt
```
Una vez realizada la instalación sin ningún error podremos ver como se ha
creado una nueva interfaz de red en el nodo de red (``nodenetwork``), llamada ``br-ex``. Ésta se puede ver ejecutando ``ifconfig -a``. Una vez comprobado
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

### Acceder a la interfaz web

Accedemos al `dashboard` escribiendo en nuestro navegador web la IP correspondiente al ``nodecontroller``.

```
https://192.168.10.150/dashboard
```

### Crear proyectos y añadir usuarios

Para crear un proyecto y añadir usuarios en OpenStack tenemos dos métodos: mediante la línea de comandos o mediante la interfaz.

Si elegimos la primera opción, tendremos que hacer ``source`` del fichero ``keystoneadminrc``. Para ello ejecutamos en ``nodecontroller`` el siguiente comando: ``source keystoneadminrc``.

Una vez hecho esto, deberemos cargar una serie de variables que servirán para autenticarnos contra OpenStack y poder ejecutar comandos propios sin problema. Así pues, añadimos el siguiente contenido a un fichero ``.sh``, lo guardamos y volvemos a hacerle ``source`` con ``source credenciales.sh``.

```
export OS_USERNAME=admin
export OS_PASSWORD=<contraseña admin>
export OS_PROJECT_NAME=<nombre proyecto>
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://nodecontroller:<port>/v3
export OS_IDENTITY_API_VERSION=3
```
En caso de que tengamos alguna duda sobre estas credenciales, nos podemos descargar uno ya creado desde nuestra interfaz web. Para ello accedemos a  Proyecto&rightarrow;Compute&rightarrow;Acceso y seguridad y ``Fichero RC de OpenStack v3 o v2`` según el caso.

Desde interfaz:
- Nos logeamos en el dashboard con las credenciales del admin y accedemos en el menú situado a la izquierda a Identidad&rightarrow;Proyectos&rightarrow; y clickamos en crear proyecto

Desde terminal:
```
openstack project create --domain default --description "Proyecto demo" demo
```

Para crear usuarios

Desde interfaz:
- Con el mismo login de admin. Vamos a Identidad&rightarrow;Usuarios&rightarrow;Crear usuario. Rellenamos los parámetros que se pidan, en mi caso solo rellenamos los campos obligatorios y asociamos el usuario al proyecto que deseemos.

Desde terminal:
```
openstack user create --domain default --password-prompt demo
```

#### Crear un "sabor" e imagen

#### Crear la red y el router del proyecto

#### Añadir grupos de seguridad y claves

### Lanzar una instancia

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



# Referencias
