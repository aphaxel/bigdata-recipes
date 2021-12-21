# Instrucciones de Instalación

Instalación de Ubuntu server 20.04 LTS en Español

## Esquema de particionamiento para la instalación:

El cluster esta compuesto de servidores que poseen

Se define el espacio para el aprovisionamiento del sistema operativo y las particiones de almacenamiento para el sistema de archivos distribuido HDFS para Hadoop

- Esquema de particonado teorico:
    - 512Mb  para espacio de UEFI /boot
    - 80Gb   para espacio de RAIZ /
    - 8Gb    para espacio SWAP
    - 20Gb   para TMP /tmp 
    - 140Gb  para espacio VAR /var

- Particion de montaje:
    - 100Gb para SHARED /shared

Se sigue el esquema de instalación asistida que provee el instalador de la distribución, el unico paquete adiccional para instalar durante la instalacion es el servidor SSH

# Instruciones configuración primer inicio

Configuración de la red con IP fija:
Primero hacer update del sistema 

```console
$ sudo apt update 
$ sudo apt upgrade
$ sudo reboot
```

### Instalar paquetes basicos y complementarios del sistema:

```console
$ sudo apt install net-tools build-essential
```
### Configurar las ip de manera fija

```console
$ sudo nano /etc/netplan/00-installer-config.yaml
```

Template de configuración:

```
# This is the network config written by 'subiquity'
network:
  bonds:
    bond0:
      interfaces:
      - enp0s3
      - enp0s8
      parameters:
        mode: active-backup
        primary: enp0s3
  ethernets:
    enp0s3: {}
    enp0s8: {}
  version: 2
  bridges:
    br0:
      addresses:
        - 192.168.1.10/24
      dhcp4: false
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
        search: []
      interfaces:
        - bond0
    enp0s9:
      dhcp4: yes
```
Aplicar los cambios:

```console
$ sudo netplan try
$ sudo netplan apply
$ sudo reboot
```
### Instalar paquetes para NFS

```console
$ sudo apt install nfs-common
```
Editar FSTAB para agregar el punto de montage de la NAS en el server

```console
$ sudo vim /etc/fstab
```
Agregar las siguientes lineas al fstab para que se realize el automontado de la unidad de almacenamiento compartido: 
```
##### EXPORT HOME #####
192.168.1.10:/home    /home                                           nfs     defaults        0 0
192.168.1.10:/shared/data  /shared/data                               nfs     defaults        0 0
```
Reiniciar el sistema: 
```console
$ sudo reboot
```
### Crear anillo de paso SSH ###

Se crean claves ssh y se crea nivel de autorizacion via **authorized_keys**

```console
$ ssh-keygen
$ cd
$ cd .ssh
$ cat id_rsa.pub >> authorized_keys
```
### crear la lista de servers registrados en el host file

```console
$ sudo nano /etc/hosts
```
Ejemplo archivo hosts 

```console
127.0.0.1 localhost
127.0.1.1 master

##### CLUSTER #####
192.168.1.10 master
192.168.1.11 worker01
192.168.1.12 worker02
172.16.10.13 worker03
172.16.10.14 worker04

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Seguimos con el proceso de instalación de los paquetes faltantes y se inicia con la configuración del cluster de HADOOP
