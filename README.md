# Instalación de OpenNebula


## Arquitectura del sistema

Para esta instalación básica usaremos 3 nodos físicos con el siguiente esquema y arquitectura de red:

<a data-flickr-embed="true" data-footer="true"  href="https://www.flickr.com/photos/manuparra/32868225721/in/dateposted-public/" title="OpenStack Basic Architecture"><img src="https://c1.staticflickr.com/4/3832/32868225721_62f7016426_b.jpg" width="777" height="550" alt="OpenStack Basic Architecture"></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

Nodos del diagrama:

- Control
- Computo: Se encarga de la gestión/ejecución, etc. de las MVs
- Red: Exclusivamente dedicado al soporte de redes en la instalación de OpenStack. Hay muchas posibilidades, pero como recomendación, lo ideal es que sea un nodo dedicado a esta tarea y no se mezcle con los demás.

Algunas consideraciones sobre el diagrama:

- En nuestro caso todos los nodos tienen instalado **CentOS7 Minimal** como distribución linux
- Todos los nodos conectados a la misma red y switch por ethernet (o infiniband)
- Todos los nodos deben tener acceso a internet
- Uno de los nodos será exclusivo e independiente para la gestión de redes (Neutron) de OpenStack.

Características de los nodos

- 32 cores
- 256 GB RAM
- 2 HD con 2TB cada uno
- 4 Tarjetas de red y 1 infiniband, pero sólo se usará una conexión ethernet.


