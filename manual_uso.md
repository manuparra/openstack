* [Acceder a la interfaz web](#acceder-a-la-interfaz-web)
* [Crear proyectos y añadir usuarios](#crear-proyectos-y-añadir-usuarios)
   * [Crear un "sabor" e imagen](#crear-un-sabor-e-imagen)
   * [Crear la red y el router del proyecto](#crear-la-red-y-el-router-del-proyecto)
   * [Añadir grupos de seguridad y claves](#añadir-grupos-de-seguridad-y-claves)
* [Lanzar una instancia](#lanzar-una-instancia)         


### Acceder a la interfaz web

Accedemos al `dashboard` escribiendo en nuestro navegador web la IP correspondiente al ``nodecontroller``.

```
https://192.168.10.150/dashboard
```
![install_004](https://github.com/manuparra/openstack/blob/master/imgs/img_004.png?raw=true)

Parece que la instalación se ha completado sin incidencias.

![install_005](https://github.com/manuparra/openstack/blob/master/imgs/img_005.png?raw=true)

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
![install_006](https://github.com/manuparra/openstack/blob/master/imgs/img_006.png?raw=true)
Desde terminal:
```
openstack project create --domain default --description "Proyecto demo" demo
```

Para crear usuarios

Desde interfaz:
- Con el mismo login de admin. Vamos a Identidad&rightarrow;Usuarios&rightarrow;Crear usuario. Rellenamos los parámetros que se pidan, en mi caso solo rellenamos los campos obligatorios y asociamos el usuario al proyecto que deseemos.
![install_007](https://github.com/manuparra/openstack/blob/master/imgs/img_007.png?raw=true)
Desde terminal:
```
openstack user create --domain default --password-prompt demo
```

#### Crear un "sabor" e imagen

Para crear un "sabor" nos dirigimos a la pestaña Admin&rightarrow;Flavors&rightarrow;
 y hacemos *click* en *Create flavor*
![install_008](https://github.com/manuparra/openstack/blob/master/imgs/img_008.png?raw=true)

Especificamos el nombre que tendrá, las VCPU, el *Root Disk*, el *Ephemeral Disk* y el *Swap*.

![install_009](https://github.com/manuparra/openstack/blob/master/imgs/img_009.png?raw=true)

Para crear una imagen nos dirigimos esta vez a Admin&rightarrow;Images&rightarrow;  y hacemos *click* en *Create image*.

Especificamos el nombre, la ruta donde tenemos dicha imagen descargada y el formato, en el caso mostrado QCOW2.

![install_010](https://github.com/manuparra/openstack/blob/master/imgs/img_010.png?raw=true)

Con esto ya tendríamos nuestra imagen disponible para ser usada.

 #### Crear la red y el router del proyecto

Para la creación de la red hay que loguearse con un usuario, por ejemplo, el que hemos creado anteriormente. Se tienen que configurar dos redes (interna y externa), la interna será la red que se encargará de asignar entre las máquinas la IP y la externa será através de la que saldrá al exterior(a Internet). Para la creación de la red interna ```Proyecto -> Redes -> Crear red```. Esta sería la configuración:
```
Nombre de la red: red interna
Estado de administracion: Arriba
Crear Subred: Marcado
Nombre de Subred: subred-interna
Direcciones de red: 10.10.0.0/24
Version IP: IPv4
IP de la Puerta de Enlace: 10.10.0.1
DHCP: Habilitado
```
La IP proporcionada en esta red puede ser aleatoria total ya que va a ser la red interna.

Para la creación de la red externa hacemos lo mismo que hemos hecho con la red interna ```Proyecto->Redes->Crear Red```

```
Nombre de la red: red externa
Estado de administracion: Arriba
Crear Subred: Marcado
Nombre de Subred: subred-externa
Direcciones de red: 192.168.10.0/24
Version IP: IPv4
IP de la Puerta de Enlace: 192.168.10.1
DHCP: Habilitado
```
Para saber la GATEWAY podemos ver el fichero ```br-ex``` que modificamos unos pasos atrás. Dado que estamos creando la red externa que será la que de acceso hacia afuera en las MVS, la puerta de enlace debe de ser la misma que la configurada en el fichero ```br-ex``` y la IP en un rango de éste.

Una vez hecho todo esto desde el usuario, tenemos que marcar la Red Externa como Externa. Nos deslogeamos del usuario y nos logeamos como admin y accedemos a ```Admin->Sistema->Redes```, pulsamos la Red Externa creada y marcamos el cuadrado de ```red externa```.

Establecida la red desde el administrador como externa, nos volvemos a logear con el usuario y pasamos a crear el router para conectar nuestras dos redes y poder enviar y recibir paquetes. Accedemos a ```Proyecto->Red->Routers```

```
Nombre del Router: <Nombre del router>
Estado de administracion: Arriba
Red Externa: red externa
```
Una vez creado el router debemos de ser capaces de verlo en el ```dashboard``` del router. Pinchamos en el router que hemos añadido y clickamos en ```Añadir interfaz```. Seleccionamos la subred interna, en nuestro caso ```subred-interna```, el resto de datos no es necesario añadirlos por lo que los dejamos en blanco y guardamos. Hecho todo esto ya estaría la red lista

#### Añadir grupos de seguridad y claves

### Lanzar una instancia
