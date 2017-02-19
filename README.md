# Instalación de OpenNebula


## Arquitectura del sistema

Para esta  instalación básica usaremos 3 nodos físicos con el siguiente esquema y arquitectura de red:



Algunas consideraciones sobre el diagrama:

- En nuestro caso todos los nodos tienen instalado **CentOS7 Minimal** como distribución linux
- Todos los nodos conectados a la misma red y switch por ethernet (o infiniband)
- Todos los nodos deben tener acceso a internet
- Uno de los nodos será exclusivo e independiente para la gestión de redes (Neutron) de OpenStack.
