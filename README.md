# openshift-setup
Setup docs for OpenShift on Atomic

# OpenShift setup

## Atomic Setup
Target Atomic hosts. We have an environment setup for usage via the [atomic-manual-setup](coming soon)
You will find full information on host IP's and build steps there.

## Pre-req
Please make sure the following requirements are met

  1. Atomic setup on **1** master and **3** nodes
  2. SSH key exchange from master to nodeX
  3. Full DNS for each master/nodes (including infra-hostname)

## Atomic Setup
Allow GlusterFS communication.

Edit the iptables file
`vi /etc/sysconfig/iptables`

add the following ***Right after `:INPUT ACCEPT [0:0]`***
```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 24007 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 24008 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2222 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m multiport --dports 49152:49664 -j ACCEPT
```

Now restart to reload iptables
`systemctl restart iptables`

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

Once the cluster is up you need to add the cluster admin role to a group.

*** Note: always use upper case groups ***

```
  oadm groups new admingrp
  oadm groups add-users admingrp user1 user2
  oadm policy add-role-to-group cluster-admin admingrp
  oadm policy add-role-to-group cluster-admin admgrp -n <namespace>
```

Add a DEMO group
```
  oadm groups new DEMO
  oadm groups add-users DEMO user3 user4 user5
```


### Notes
To List groups and their membership
```
  oc get groups
  oc get group admingrp
```

To remove membership to a group
```
  oadm groups remove-users admingrp FOOBAR
```

To remove a group
```
  oc delete group FOOBAR
```

If you logout by accident on the master
```
  oc login -u system:admin
```

### Setting up wildcard SSL

```
oc export secret router-certs -o yaml > router-certs.backup.$date.yaml
mv router-certs.backup..yaml router-certs.backup.oct12.yaml
oc delete secret router-certs
oc secrets new router-certs tls.crt=master.server.crt tls.key=star_nonprod_apps_novascotia_ca.key --type='kubernetes.io/tls' --confirm
oc rollout latest
oc get ods
oc logs router-pod
```

#### Containerized Install
RedHat's official notes on [Containerized Install](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.6/html-single/release_notes/#ocp-36-installation
2.3.7.6)

#### Ansible Inventory
Take a look at how to make an [Inventory File](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.6/html-single/installation_and_configuration/#adv-install-example-inventory-files)

#### Authentication
[HTPasswd setup](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.6/html/installation_and_configuration/install-config-configuring-authentication#HTPasswdPasswordIdentityProvider)
Ensure the format is MD5 *NOT CRYPT*

[GitLab setup](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.6/html/installation_and_configuration/install-config-configuring-authentication#GitLab)
