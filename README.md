# CLUSTER PACEMAKER / COROSYNC EN AWS EC2

> [!IMPORTANT]
> POC.
> Work more than fine. But be carefull in production. 
> There is a lot of improvements to do.

## DESCRIPCION
Basic recipe to install a two nodes Pacemaker / Corosyc Cluster within AWS EC2 with a VIP able to move between zones

**Based on [this article](https://aws.amazon.com/blogs/apn/making-application-failover-seamless-by-failing-over-your-private-virtual-ip-across-availability-zones/)**

## Packages
Cluster packages.
```bash
apt install pacemaker corosync pcs
```
Package to configure STONISH on AWS EC2
```bash
apt install fence-agents-aws
```

## Corosync configuration
Basic Corosyn configuration.

### authkey configuration
Run **corosync-keygen** on nodo1 (The command deliver the auth file at de rigth path.)
Copy to the node2

> [!CAUTION]
> Permision need to be 400

```bash
root@adam:/etc/corosync# corosync-keygen
Corosync Cluster Engine Authentication key generator.
Gathering 2048 bits for key from /dev/urandom.
Writing corosync key to /etc/corosync/authkey.
```

### Corosync (cluster) configuration

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
> Values of name attribute need to be resolved by **DNS** o **/etc/hosts** file on both nodes.

## SecurityGroups configuration
Open UDP ports:
- 5404
- 5405

Open TCP port:
- 2224

## STONISH Mechanism configuration
Mechanism to immediately kill a node in case of quorum discrepancies. Critical to avoid the feared "split brain".

> [!IMPORTANT]
> Fisrt singularity of this cluster on AWS EC2. STONISH between is here a sofware with access to AWS EC2 API and so is able to shotdown the instance. 

### Requisitos.

The package **fence-agents-aws** insttaled and aws cli insttaled and configured with this permisions for the user whose credentials are used by aws cli.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceStatus",
                "ec2:StopInstances",
                "ec2:TerminateInstances"
            ],
            "Resource": "*"
        }
    ]
}
```

### Creating resources
Creating resources

```bash
root@adam: pcs stonith create aws-fence-adam fence_aws region=eu-west-1 access_key=ACCESS_KEY secret_key=SECRET_KEY plug=ID_INSTANCIA_NODE_1
root@adam: pcs stonith create aws-fence-eva fence_aws region=eu-west-1 access_key=ACCESS_KEY secret_key=SECRET_KEY plug=ID_INSTANCIA_NODE_2
```

Creating constraints to be sure that any fence runs only on its node

```bash
root@adam: pcs constraint location aws-fence-adam prefers adam=INFINITY
root@adam: pcs constraint location aws-fence-eva prefers eva=INFINITY
  ```

## Configuration the resource AWS VIP

### Pre Steps

- Be sure the nodes have [sourcedestcheck](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-networkinterface.html#cfn-ec2-networkinterface-sourcedestcheck)

- Add this new IAM permissions to the user whose credentials are used by aws cli

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateRoute",
                "ec2:DeleteRoute"
            ],
            "Resource": "arn:aws:ec2:*:*:route-table/*"
        }
    ]
}
```

> [!NOTE]
> [awsvip](https://github.com/ClusterLabs/resource-agents/blob/main/heartbeat/awsvip) y [aws-vpc-move-ip](https://github.com/ClusterLabs/resource-agents/blob/main/heartbeat/aws-vpc-move-ip) were imposible to use. I'm not sure why.


### Preparing the custom resource
> [!WARNING]
> Nexts steps need to be done on both nodes

Create a folder into the resources structure

```bash
root@adam: mkdir /usr/lib/ocf/resource.d/cicely
```

Create a file named **vip** on **usr/lib/ocf/resource.d/cicely**

```
#!/bin/bash

# Script de recurso personalizado para Pacemaker

TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 3600")
MAC=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/network/interfaces/macs/ | head -n 1 | tr -d '/')
ENI=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/network/interfaces/macs/${MAC}/interface-id)

: ${OCF_ROOT=/usr/lib/ocf}
: ${OCF_FUNCTIONS_DIR=${OCF_ROOT}/lib/heartbeat}
. ${OCF_FUNCTIONS_DIR}/ocf-shellfuncs

OCF_RESKEY_ip_default=""
OCF_RESKEY_routing_table_default=""
OCF_RESKEY_interface_default="ens5"

: ${OCF_RESKEY_ip=${OCF_RESKEY_ip_default}}
: ${OCF_RESKEY_routing_table=${OCF_RESKEY_routing_table_default}}
: ${OCF_RESKEY_interface=${OCF_RESKEY_interface_default}}

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
    <parameter name="routing_table" required="1">
      <longdesc lang="en">Name of the routing table(s), where the route for the IP address should be changed. If declaring multiple routing tables they should be separated by comma. Example: rtb-XXXXXXXX,rtb-YYYYYYYYY</longdesc>
      <shortdesc lang="en">routing table name(s)</shortdesc>
      <content type="string" default="${OCF_RESKEY_routing_table_default}" />
    </parameter>
    <parameter name="interface" required="0">
      <longdesc lang="en">Name of the network interface, i.e. eth0</longdesc>
      <shortdesc lang="en">network interface name</shortdesc>
      <content type="string" default="${OCF_RESKEY_interface_default}" />
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
        # Deleting route
        aws ec2 delete-route \
        --route-table-id ${OCF_RESKEY_routing_table} \
        --destination-cidr-block ${OCF_RESKEY_ip}/32

        sleep 3

        # Creating route
        aws ec2 create-route \
        --route-table-id ${OCF_RESKEY_routing_table} \
        --destination-cidr-block ${OCF_RESKEY_ip}/32 \
        --network-interface-id ${ENI}

        sleep 3

        ip addr add ${OCF_RESKEY_ip}/32 dev $OCF_RESKEY_interface

        sleep 3

        if [ $? -eq 0 ]; then
                return $OCF_SUCCESS
        else
                return $OCF_ERR_GENERIC
        fi
}

stop() {
        # Deleting route
        aws ec2 delete-route \
        --route-table-id ${OCF_RESKEY_routing_table} \
        --destination-cidr-block ${OCF_RESKEY_ip}/32

        sleep 3

        ip addr del ${OCF_RESKEY_ip}/32 dev $OCF_RESKEY_interface

        sleep 3

        if [ $? -eq 0 ]; then
                return $OCF_SUCCESS
        else
                return $OCF_ERR_GENERIC
        fi
}

monitor() {
        # Super simple way to monitor de VIP
         ip addr show | grep $OCF_RESKEY_ip

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

Give execution permisions to the file

```bash
root@adam: chmod 775 /usr/lib/ocf/resource.d/cicely/vip
```

### Creating the final resource

```bash
root@adam: spcs resource create vip ocf:fluzo:vip ip=10.1.1.10 routing_table=rtb-00000000000000

```

# TODO (Ordered by priority)
- Improve logs
- Improve error management
- Improve the monitor system.
- [Don't repeat yourself (DRY)](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) or **duplication is evil**
