

# Añadir imagen de SO con Fedora23 con cloud-init

Autenticamos con el fichero ``keystonerc_admin``:

```
source keystonerc_admin
```

Verificamos la lista de imagenes:

```
openstack image list
```

Descargamos Fedora 23:

```
curl -O http://ftp.yzu.edu.tw/Linux/Fedora/linux/releases/23/Cloud/x86_64/Images/Fedora-Cloud-Base-23-20151030.x86_64.qcow2
```

Añadirmos la imagen al repositorio de OpenStack:

```
openstack image create --disk-format qcow2 --container-format bare --public --file ./Fedora-Cloud-Base-23-20151030.x86_64.qcow2 fedora23-image
```


