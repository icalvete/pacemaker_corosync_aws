# CLUSTER PACEMAKER / COROSYNC EN AWS EC2

> [!IMPORTANT]
> ESTO ES UNA POC. No es una receta para poner en produccion.
> Pero funciona. Se ha probado con existo y se puede perfeccionar mucho.

> [!IMPORTANT]
> Esta solucion solo funciona si las maquinas estan en la misma subnet

## DESCRIPCION
Receta basica para instalar y configurar un cluster pacemaker / corosync en AWS EC2 en un cluster de dos nodos

Contiene un unico recurso. Una VIP que se desplaza entre los nodos a voluntad o en caso de fallo de uno de ellos.
**Todo se basa en mover una ENI de una instancia a otra.**

## Paquetes
Paquetes necesario basicos para ejecutar y configuar el cluster.
```bash
apt install pacemaker corosync pcs
```
Paquete para configurqar el STONISH en AWS EC2
```bash
apt install fence-agents-aws
```

## Configuracion de corosync
Configuracion basica de corosync

### Configuracion authkey
Crear el fichero con corosync-keygen en en nodo1 (crea el fichero en su ubicacion correcta)
Luego copiarlo al otro nodo.

> [!CAUTION]
> Los permisos tienen que ser 400.

```bash
root@adam:/etc/corosync# corosync-keygen
Corosync Cluster Engine Authentication key generator.
Gathering 2048 bits for key from /dev/urandom.
Writing corosync key to /etc/corosync/authkey.
```

### Configuracion de corosync (cluster)

```bash
# Please read the corosync.conf.5 manual page
system {
	# This is required to use transport=knet in an unprivileged
	# environment, such as a container. See man page for details.
	allow_knet_handle_fallback: yes
}

totem {
	version: 2

	# Corosync itself works without a cluster name, but DLM needs one.
	# The cluster name is also written into the VG metadata of newly
	# created shared LVM volume groups, if lvmlockd uses DLM locking.
        cluster_name: cicely

	# crypto_cipher and crypto_hash: Used for mutual node authentication.
	# If you choose to enable this, then do remember to create a shared
	# secret with "corosync-keygen".
	# enabling crypto_cipher, requires also enabling of crypto_hash.
	# crypto works only with knet transport
	crypto_cipher: none
	crypto_hash: none
    	secauth: on
        transport: udpu
	auth: off
}

logging {
	# Log the source file and line where messages are being
	# generated. When in doubt, leave off. Potentially useful for
	# debugging.
	fileline: off
	# Log to standard error. When in doubt, set to yes. Useful when
	# running in the foreground (when invoking "corosync -f")
	to_stderr: yes
	# Log to a log file. When set to "no", the "logfile" option
	# must not be set.
	to_logfile: yes
	logfile: /var/log/corosync/corosync.log
	# Log to the system log daemon. When in doubt, set to yes.
	to_syslog: yes
	# Log debug messages (very verbose). When in doubt, leave off.
	debug: off
	# Log messages with time stamps. When in doubt, set to hires (or on)
	#timestamp: hires
	logger_subsys {
		subsys: QUORUM
		debug: off
	}
}

quorum {
	# Enable and configure quorum subsystem (default: off)
	# see also corosync.conf.5 and votequorum.5
	provider: corosync_votequorum
        expected_votes: 2
        two_node: 1
        wait_for_all: 1
}

nodelist {
	# Change/uncomment/add node sections to match cluster configuration

	node {
		# Hostname of the node.
		name: adam
		# Cluster membership node identifier
		nodeid: 1
		# Address of first link
		ring0_addr: 192.168.1.1
	}

	node {
		# Hostname of the node.
		name: eva
		# Cluster membership node identifier
		nodeid: 2
		# Address of first link
		ring0_addr: 192.168.1.2
	}
}
```
> [!CAUTION]
> los valores de atributo name de la seccion nodelist deben poderse resover via **DNS** o **/etc/hosts** file en los dos nodos.

## Configuracion de SecurityGroups
Abrir puertos UDP:
- 5404
- 5405

Abrir puerto TCP
- 2224

## Configuracion del mecanismo de STONISH
Ese mecanismo se encarga de matar de manera fulminante un nodo en caso de discrepancias de quorum y es critico para evitar el temido "split brain"

> [!IMPORTANT]
> Esta es la primera de las diferencias de un cluster fuera de AWS EC2. Aqui en mecanismo de STONISH para por usar un producto que da acceso al API de AWS EC2 y de esta forma pedir un shootdown de la instancia afectada.

### Requisitos.
El paquete **fence-agents-aws** instalado y el aws cli instalado y configurado con unas credencias con estos permisos.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:StopInstances",
                "ec2:StartInstances",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DescribeInstances",
                "ec2:AttachNetworkInterface",
                "ec2:DetachNetworkInterface",
                "ec2:AssignPrivateIpAddresses",
                "ec2:UnassignPrivateIpAddresses",
                "ec2:ModifyNetworkInterfaceAttribute"
            ],
            "Resource": "*"
        }
    ]
}
```

### Creacion de los recursos
Se crean los recursos

```bash
root@adam: pcs stonith create aws-fence-adam fence_aws region=eu-west-1 access_key=ACCESS_KEY secret_key=SECRET_KEY plug=ID_INSTANCIA_NODE_1
root@adam: pcs stonith create aws-fence-eva fence_aws region=eu-west-1 access_key=ACCESS_KEY secret_key=SECRET_KEY plug=ID_INSTANCIA_NODE_2
```

Se crean las constraints para asegurarse que cada fence se ejecuta solo en su nodo.

```bash
root@adam: pcs constraint location aws-fence-adam prefers adam=INFINITY
root@adam: pcs constraint location aws-fence-eva prefers eva=INFINITY
  ```

## Configuracion del recursos AWS VIP
Para resolver el problema de crear un recursos VIP (Virtual IP) en AWS se va a seguir la siguiente estrategia

- Crear una [ENI](https://docs.aws.amazon.com/es_es/AWSEC2/latest/UserGuide/using-eni.html)
- Añadirla a una de las instancias
- Apuntar la IP
- Apuntar el id de la [ENI](https://docs.aws.amazon.com/es_es/AWSEC2/latest/UserGuide/using-eni.html)

Con esos datos se crear un recurso custom que mueve la [ENI](https://docs.aws.amazon.com/es_es/AWSEC2/latest/UserGuide/using-eni.html) de una instancia a otra

> [!NOTE]
> Se han probado [awsvip](https://github.com/ClusterLabs/resource-agents/blob/main/heartbeat/awsvip) y [aws-vpc-move-ip](https://github.com/ClusterLabs/resource-agents/blob/main/heartbeat/aws-vpc-move-ip) pero no encajan del todo y ademas tienen bastantes bugs


### Preparando el recurso custom
> [!WARNING]
> Las siguientes instrucciones se ejecutan en los dos nodos.

Se crea una carpeta para la organizacion dentro de la estructura de recursos

```bash
root@adam: mkdir /usr/lib/ocf/resource.d/cicely
```

Se crea un fichero con el nombre **vip** en **/usr/lib/ocf/resource.d/cicely**

```
#!/bin/bash

# Script de recurso personalizado para Pacemaker

: ${OCF_ROOT=/usr/lib/ocf}
: ${OCF_FUNCTIONS_DIR=${OCF_ROOT}/lib/heartbeat}
. ${OCF_FUNCTIONS_DIR}/ocf-shellfuncs

OCF_RESKEY_ip_default=""
OCF_RESKEY_eni_default=""
: ${OCF_RESKEY_ip=${OCF_RESKEY_ip_default}}
: ${OCF_RESKEY_eni=${OCF_RESKEY_eni_default}}

TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 3600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)

meta_data() {
    cat <<END
<?xml version="1.0"?>
<!DOCTYPE resource-agent SYSTEM "ra-api-1.dtd">
<resource-agent name="mi-recurso-personalizado">
  <version>1.0</version>
  <longdesc lang="en">
  Resource Agent to move IP addresses within a VPC of the Amazon Webservices EC2
  by changing an detach / attach an ENI

  Credentials needs to be setup by running "aws configure", or by using AWS Policies.

  See https://aws.amazon.com/cli/ for more information about awscli.
</longdesc>
<shortdesc lang="en">Move IP within a VPC of the AWS EC2</shortdesc>
<parameters>
  <parameter name="ip" required="1">
    <longdesc lang="en">VPC private IP address</longdesc>
    <shortdesc lang="en">VPC private IP</shortdesc>
    <content type="string" default="${OCF_RESKEY_ip_default}" />
  </parameter>
  <parameter name="eni" required="1">
    <longdesc lang="en">ENI that provide the IP</longdesc>
    <shortdesc lang="en">ENI</shortdesc>
    <content type="string" default="${OCF_RESKEY_eni_default}" />
  </parameter>
</parameters>
<actions>
  <action name="start" timeout="20s"/>
  <action name="stop" timeout="20s"/>
  <action name="monitor" timeout="20s" interval="10s"/>
  <action name="meta-data" timeout="5s"/>
  </actions>
</resource-agent>
END
}

start() {
    # Getting the Attach ID
    attachment_id=$(aws ec2 describe-network-interfaces --network-interface-ids $OCF_RESKEY_eni --query 'NetworkInterfaces[0].Attachment.AttachmentId' --output text)

    # If Attach ID, detach the ENI from whereever is.
    if [ "$attachment_id" == "None" ]; then
      echo "La variable tiene el valor None"
    else
      aws ec2 detach-network-interface --attachment-id $attachment_id
      sleep 15
    fi

    # Attach the ENI to this node instance.
    aws ec2 attach-network-interface --network-interface-id $OCF_RESKEY_eni --instance-id $INSTANCE_ID --device-index 1
    sleep 10

    if [ $? -eq 0 ]; then
        return $OCF_SUCCESS
    else
        return $OCF_ERR_GENERIC
    fi
}

stop() {
    # Getting the Attach ID
    attachment_id=$(aws ec2 describe-network-interfaces --network-interface-ids $OCF_RESKEY_eni --query 'NetworkInterfaces[0].Attachment.AttachmentId' --output text)

    # If Attach ID, detach the ENI from whereever is.
    if [ "$attachment_id" == "None" ]; then
      echo "La variable tiene el valor None"
    else
      aws ec2 detach-network-interface --attachment-id $attachment_id
      sleep 15
    fi

    if [ $? -eq 0 ]; then
        return $OCF_SUCCESS
    else
        return $OCF_ERR_GENERIC
    fi
}

monitor() {
    # Super simple way to monitor de VIP
    ifconfig | grep $OCF_RESKEY_ip

    if [ $? -eq 0 ]; then
        return $OCF_SUCCESS
    else
        return $OCF_NOT_RUNNING
    fi
}

case $__OCF_ACTION in
    meta-data)
        meta_data
        exit $OCF_SUCCESS
        ;;
    start)
        start
        exit $?
        ;;
    stop)
        stop
        exit $?
        ;;
    monitor)
        monitor
        exit $?
        ;;
    *)
        echo "Uso: $0 {start|stop|monitor|meta-data}"
        exit $OCF_ERR_UNIMPLEMENTED
        ;;
esac
```

El valor de **ENI_ID** debe ser el id de la ENI que se creo en los pasos previos.
El valor de **VIP** debe ser el valor que de la IP que se le concede a la ENI anterior

> [!IMPORTANT]
> El valor de **INSTANCE_ID** se obtiene usando los [metadatos de la instancia](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)

Se dan permisos de ejecucion al fichero

```bash
root@adam: chmod 7775 /usr/lib/ocf/resource.d/cicely/vip
```

### Creando el recurso en el cluster

```bash
root@adam: pcs resource create vip ocf:cicely:vip op start timeout=60 op stop timeout=60 op monitor interval=10  start-delay=30s
```

#TODO (Ordered by priority)
- Improve the monitor system.
- [Don't repeat yourself (DRY)](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) or **duplication is evil**
- Allow instances in different subnets
