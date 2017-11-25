## Install OCP 3.6 on Red Hat Atomic Host

This guide installs OCP 3.6 on Red Hat Atomic Host using the ansible-container installer, Gluster Container Native Storage and LDAP authentication. An external LB configured to use SSL passthru is needed in front of the infrastructure and master nodes.

The inventory sets up a basic HA environment with 3 masters, 3 infrastructure nodes, 4 application nodes and 3 storage nodes.

> [Red Hat Atomic Host VMware Setup](atomic-setup.md)

> [Inventory file](inventory)

## Atomic OS Setup

### On the storage nodes
Allow GlusterFS communication.

Edit the iptables file
```
vi /etc/sysconfig/iptables
```

add the following ***Right after `:INPUT ACCEPT [0:0]`***
```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 24007 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 24008 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2222 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m multiport --dports 49152:49664 -j ACCEPT
```

Now restart to reload iptables
`systemctl restart iptables`

### On the install source node

Generate roots public key and then distribute it to each OCP node
```
ssh-keygen -t rsa
ssh-copy-id root@node1
OR
cat ~/.ssh/id_rsa.pub | ssh root@node1 "mkdir -p ~/.ssh && cat >>  ~/.ssh/authorized_keys"
```

Download the latest Ansible installer image
```
atomic pull --storage ostree registry.access.redhat.com/openshift3/ose-ansible:v3.6
```

Install the container with a inventory file
```
atomic install --system \
    --storage=ostree \
    --name=openshift-installer \
    --set INVENTORY_FILE=/root/inventory \
    registry.access.redhat.com/openshift3/ose-ansible:v3.6 
    systemctl start openshift-installer
```

## Post Install Steps

Make the glusterfs storage class the default. Out of the box it is not:

```
# oc get storageclass
NAME                TYPE
glusterfs-storage   kubernetes.io/glusterfs
```

Make it the default:

```
oc patch storageclass glusterfs-storage -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}' -n openshift
```

Verify:

```
# oc get storageclass
NAME                          TYPE
glusterfs-storage (default)   kubernetes.io/glusterfs
```

Add the cluster admin role to a group.

*** Note: always use upper case groups ***

```
  oadm groups new MYGROUP
  oadm groups add-users MYGROUP ADMIN1 ADMIN2
  oadm policy add-role-to-group cluster-admin MYGROUP
```


### Setting up wildcard SSL for your Infrastructure nodes

Concatenate the wildcard and root/intermediate certs:
```
cat star_domain_tld.crt Intermediate_CA.crt Root_CA.crt > master.server.crt
```

Export the existing cert:
```
oc export secret router-certs -o yaml > router-certs.backup.yml
```
Delete existing secret
```
oc delete secret router-certs
```
Install the new cert and deploy it across the cluster
```
oc secrets new router-certs tls.crt=master.server.crt tls.key=star_domain_tld.key --type='kubernetes.io/tls' --confirm
oc rollout latest dc/router
oc get pods
oc logs dc/router
```
