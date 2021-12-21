----------------------------------------------------------------------
# CONFIGURACIÓN DEL CLUSTER HADOOP
----------------------------------------------------------------------

## HADOOP + HDFS
----------------------------------------------------------------------
### 1. En cada equipo crear usuario y grupo hadoop

```
sudo useradd -d /home/hadoop -m hadoop
sudo passwd hadoop
sudo usermod -a -G sudo hadoop
sudo user mod -s /bin/bash hadoop
```

Reiniciar e ingresar como usuario hadoop

----------------------------------------------------------------------

### 2. Se debe garantizar un esquema de nombres en los archivos hosts de cada maquina del cluster:

```
sudo nano /etc/hostname —> cambiar nombre según corresponda: master o slave01 
sudo nano /etc/hosts —> adicionar IP y nombre de equipo
```
Host file: agregar al final 
```
##### Big Data Network #####
192.168.1.10 master
192.168.1.11 worker01
192.168.1.12 worker02
172.16.10.13 worker03
172.16.10.14 worker04
```

Verificar con ping a cada nodo del cluster:

```
ping IP o Nombre_Equipo
```

----------------------------------------------------------------------

### 3. Configurar acceso por ssh sin contraseña
<b>NOTA:</b> Omitir este paso si se realizo en alguna fase previa y pasar al siguiente.

Instalar ssh para cliente y servidor
```
sudo apt-get install openssh-client openssh-server
sudo service ssh restart
```
Verificar el firewall, sin embargo ya que el cluster se encuentra sobre un firewall real dedicado el firewall de Ubuntu por defecto se encuentra desabilitado: 
```
sudo ufw status
- salida:
Status: inactive
```
De no ser así desactivarlo:
```
sudo ufw disable
```
Generar claves: pública y privada
```
sudo ssh-keygen -t rsa
```
Copiar la clave pública en cada nodo: 
```
sudo ssh-copy-id hadoop@worker01
sudo ssh-copy-id hadoop@worker02
...
```
Otorgar permisos en cada equipo
```
sudo chmod 700 ~/.ssh
sudo chmod 600 ~/.ssh/authorized_keys
```

Adicionalmente en master otorgar permisos
```
sudo chown `whoami` $HOME/.ssh/authorized_keys
```
Probar acceso sin contraseña por ssh desde el master hacia los slaves
```
ssh localhost
ssh hadoop@worker01
...
```
----------------------------------------------------------------------

### 4. Desactivar IPv6
```
sudo nano /etc/sysctl.conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```
Reiniciar y probar con el comando
```
sudo cat /proc/sys/net/ipv6/conf/all/disable_ipv6
```
----------------------------------------------------------------------

### 5. Instalar y configurar Java 11
```
tar xzf jdk-11.0.2_linux-x64_bin.tar.gz
cd /opt/jdk-11.0.2
```
Crear enlace simbolico a jdk11
```
ln -s jdk-11.0.2 jdk11
```
Las versiones de java por defecto no se modifican sino que son gestinadas via <b>module-enviroment</b> pueden existir varias instalaciones en cluster, para cargar alguna versión de JAVA en especial usar el comando:
```
module load java/Oracle-java-11.0.2
```
En caso de no existir crear el script para module:
```
sudo vim /shared/data/software/calima-modules/java/Oracle-java-11.0.2
```
```script
#%Module1.0#####################################################################
##
## modules Oracle Java
##
## modulefiles/java/Oracle-java-11.0.2 Written by Miguel.guevara
##
proc ModulesHelp { } {
        global version modroot

        puts stderr "\n\tConfigure JAVA global variables to system"
}

module-whatis   "Sets the environment for using Oracle JAVA JDK version"

# for Tcl script use only
set     topdir          /opt/jdk-11.0.2

set     version         11.0.2
set     sys             linux86-64

setenv          JAVA_HOME        $topdir/
prepend-path    PATH             $topdir/bin
prepend-path    MANPATH          $topdir/man
prepend-path    LD_LIBRARY_PATH  $topdir/lib
append-path     CPATH            $topdir/include
```
Probar el script de module y la version de java carga correctamente:

```
module load java/Oracle-java-11.0.2
java -version
```
Salida:
```
java version "11.0.2" 2019-01-15 LTS
Java(TM) SE Runtime Environment 18.9 (build 11.0.2+9-LTS)
Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.2+9-LTS, mixed mode)
```

----------------------------------------------------------------------

### 6. Instalar y configurar Hadoop
```
tar -xzf hadoop-3.2.1.tar.gz
```
copiar la carpeta a /opt
```
sudo mv $HOME/hadoop-3.2.1.tar.gz /opt/hadoop
sudo chown -R hadoop:hadoop /opt/hadoop/
```
Incluir variables de entorno de hadoop en un script de <b>module-enviroment</b> para que sean gestionadas vía el comando <b>module</b>:
```
vim /shared/data/software/cluster-modules/java/Oracle-java-11.0.2
```
Pegar el contenido del script: 
```script
#%Module1.0#####################################################################
##
## modules Oracle Java
##
## modulefiles/java/Oracle-java-11.0.2 Written by Miguel.guevara
##
proc ModulesHelp { } {
        global version modroot

        puts stderr "\n\tConfigure JAVA global variables to system"
}

module-whatis   "Sets the environment for using Oracle JAVA JDK version"

# for Tcl script use only
set     topdir          /opt/jdk-11.0.2

set     version         11.0.2
set     sys             linux86-64

setenv          JAVA_HOME        $topdir/
prepend-path    PATH             $topdir/bin
prepend-path    MANPATH          $topdir/man
prepend-path    LD_LIBRARY_PATH  $topdir/lib
append-path     CPATH            $topdir/include

```
En el nodo master configurar los siguientes archivos:
```
sudo nano /opt/hadoop/etc/hadoop/core-site.xml
```
```
<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://master:9000</value>
                <description>NameNode URI</description>
        </property>
</configuration>
```
```
sudo nano /opt/hadoop/etc/hadoop/hdfs-site.xml
```
```
<configuration>
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:/opt/hadoop/hadoop_data/hdfs/namenode</value>
        </property>

        <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:/opt/hadoop/hadoop_data/hdfs/datanode</value>
        </property>

        <property>
                <name>dfs.replication</name>
                <value>1</value>
        </property>
</configuration>
```
```
sudo nano /opt/hadoop/etc/hadoop/mapred-site.xml
```
```
<configuration>
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
                <description>Execution framework</description>
        </property>
</configuration>
```
```
sudo nano /opt/hadoop/etc/hadoop/yarn-site.xml
```
```
<configuration>
<!-- Site specific YARN configuration properties -->
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
                <description>shuffle service for MapReduce</description>
        </property>
        <property>
                <name>yarn.resourcemanager.resource-tracker.address</name>
                <value>master:8025</value>
                <description>ResourceManager for NodeManagers</description>
        </property>
        <property>
                <name>yarn.resourcemanager.scheduler.address</name>
                <value>master:8030</value>
                <description>ResourceManager for ApplicationMasters</description>
        </property>
        <property>
                <name>yarn.resourcemanager.address</name>
                <value>master:8050</value>
                <description>ResourceManager for clients to submit jobs</description>
        </property>
</configuration>
```
Se agregan los nombres de los nodos en el archivo slaves de hadoop
```
sudo nano /opt/hadoop/etc/hadoop/slaves
worker01
worker02
worker03
worker04
```
Se crea el directorio para almacenar los datos HDFS del NameNode
```
sudo mkdir -p /shared/hadoop/hdfs/namenode
sudo chown -R hadoop:hadoop /opt/hadoop
```
Se duplica la configuración de hadoop en todos los nodos (previamente hadoop debe estar instalado)
```
sudo scp -r /opt/hadoop/hadoop/* hadoop@worker01:/opt/hadoop/etc/hadoop
sudo chown -R hadoop:hadoop /opt/hadoop
```
Se crea el directorio en cada nodo para almacenar HDFS del DataNode
```
sudo mkdir -p /shared/hadoop/hdfs/datanode
sudo chown -R hadoop:hadoop /opt/hadoop
```
Se formatea el filesystem distribuido
```
sudo hdfs namenode -format -force
```
Se elimina el modo seguro
```
hadoop dfsadmin -safemode leave
```
Se borran los archivos del filesystem
```
hadoop fsck / -delete
```
Para iniciar hdfs y yarn
```
start-yarn.sh
start-dfs.sh
```
Se verifica que los servicios estén arriba
```
jps
```
```
NameNode
Jps
SecondaryNameNode
NodeManager
ResourceManager
DataNode
```
Se puede verificar también vía Web
- http://master:9870
- http://master:8080





----------------------------------------------------------------------
### Referencias:
- [1] http://adictosaltrabajo.com/2014/02/25/hadoop-first-steps/
- [2] http://alibabacloud.com/blog/how-to-setup-hadoop-cluster-ubuntu-16-04_593808
- [3] http://cetatech.ceta-ciemat.es/2015/03/desplegando-un-cluster-distribuido-con-hadoop-2-6-0
