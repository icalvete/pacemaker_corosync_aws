# CLUSTER PACEMAKER / COROSYNC EN AWS EC2

> [!IMPORTANT]
> ESTO ES UNA POC. No es una receta para poner en produccion.

## DESCRIPCION
Receta basica para instalar y configurar un cluster pacemaker / corosync en AWS EC2 en un cluster de dos nodos

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
        cluster_name: fluzo_logs

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
		ring0_addr: 192.168.1.11
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

## configuracion del recursos AWS VIP
Para resolver el problema de crear un recursos VIP (Virtual IP) en AWS se va a seguir la siguiente estrategia

- Crear una [ENI](https://docs.aws.amazon.com/es_es/AWSEC2/latest/UserGuide/using-eni.html)
- Añadirla a una de las instancias
- Apuntar la IP
- Apuntar el id de la [ENI](https://docs.aws.amazon.com/es_es/AWSEC2/latest/UserGuide/using-eni.html)

Con esos datos se crear un recurso custom que mueve la [ENI](https://docs.aws.amazon.com/es_es/AWSEC2/latest/UserGuide/using-eni.html) de una instancia a otra

> [!NOTE]
> Se han probado [awsvip](https://github.com/ClusterLabs/resource-agents/blob/main/heartbeat/awsvip) y [aws-vpc-move-ip](https://github.com/ClusterLabs/resource-agents/blob/main/heartbeat/aws-vpc-move-ip) pero no encajan del todo y ademas tienen bastantes bugs



