# Step-By-Step Backup and Restore Guide 
by:  *S&N AG, Paderborn, Germany*

This guide has been developed due to the lack of information about which files are important when making a Backup / Restore of OpenShift Origin. The procedure has been tested with OpenShift Origin v. 3.6 and is based on the information provided by the official documentation: https://docs.okd.io/3.6/admin_guide/backup_restore.html



## Backup
Steps 1-3 on Master-Node

Step 3 on Master + Workers

#### 1. backup certificates/keys from Master-Node:

> cd /etc/origin/master

> tar cf /tmp/certs-and-keys-$(hostname).tar *.key *.crt

  1.1 backup ca.crt and ca.key seperately, for use in restore-procedure

> cp /etc/origin/master/ca.crt <backup-dir>

> cp /etc/origin/master/ca.key <backup-dir>

#### 2. ETCD - backup  ( ETCD stores configurations and user/project-specific data   )

> sudo systemctl stop etcd

> etcdctl backup --data-dir /var/lib/etcd -backup-dir /var/lib/etcd.bak

  2.1 backup db-file

> cp /var/lib/etcd/member/snap/db /var/lib/etcd.bak/member/snap/db

  2.2 restart ETCD

> systemctl restart etcd

#### 3. backup docker-registry - certificates [if exist] (Master + Worker)

> cd /etc/docker/certs.d/

> tar cf /tmp/docker-registry-certs-$(hostname).tar *

 

 

## Restore
#### 1. reinstall OpenShift (reset Nodes, complete reinstall)

Include the following line in your Ansible-Inventory file, in order to restore the ca.crt and ca.key files.
> openshift_master_ca_certificate={'certfile': '<backup-dir>/ca.crt', 'keyfile': '<backup-dir>/ca.key'}

#### 2. restore certifikate and keys (Master-Node)

> cd /etc/origin/master

> tar xvf /tmp/certs-and-keys-$(hostname).tar

#### 3. restore docker-registry - certificates

#### 4. restore ETCD-backup (Master-Node)

> mv /var/lib/etcd /var/lib/etcd.orig

> cp -Rp $ETCD_DATA_DIR.bak /var/lib/etcd

> chown -R etcd:etcd /var/lib/etcd

  4.1 change the following file, then restart ETCD:

> vi /usr/lib/systemd/system/etcd.service


In the line containing 'ExecStart=/bin/bash  .....' , add:
> --force-new-cluster

e.g. :
> ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd --force-new-cluster   ......"


> systemctl daemon-reload

> systemctl start etcd


  4.2 revert the previous changes, then restart ETCD

> vi /usr/lib/systemd/system/etcd.service

remove the "--force-new-cluster" - option 

> systemctl daemon-reload

> systemctl restart etcd


#### 5. redeploy the ca.crt and ca.key, using the Ansible - "redeploy-certificates.yml" - playbook



  
  5.1 include the following lines in your Ansible-Inventory file:



> openshift_master_ca_certificate={'certfile': '<backup-dir>/ca.crt', 'keyfile': '<backup-dir>/ca.key'}

> openshift_master_overwrite_named_certificates=true

  5.2 run the Ansible-playbook

> ansible-playbook -i /path/to/inventory-file openshift-ansible/playbooks/byo/openshift-cluster/redeploy-certificates.yml

 
--> OpenShift should now be restored

