# Lab: Introduction to Ceph

In this lab we will deploy a Ceph cluster from scratch using only the basic tools available from Ceph (no deployment utility). This is the same way that Ceph is deployed today at DigitalOcean with [storage-cm](https://github.internal.digitalocean.com/digitalocean/storage-cm)'s [ceph-deploy playbook](https://github.internal.digitalocean.com/digitalocean/storage-cm/blob/master/playbooks/common/ceph_deploy.yml).

Note: Ceph uses cephx for authentication and authorization. cephx is not part of this training, you can read about cephx purpose and use [here](https://docs.ceph.com/en/pacific/architecture/#high-availability-authentication).

## Setting up the environment

Clone the following repository: `$ git clone git@github.internal.digitalocean.com:amarangone/ceph-tf.git`

In order to run terraform and perform this lab you will need the following:
 - Terraform installed (v1.2.9 is the tested version)
 - A DO token, it can be generated from the Cloud UI in the "API" tab
 - `doctl` configured with the same token
 - A SSH key added to your personal DO cloud account

Once you have ensured that you have all the pre-requisite, run the following from the `ceph-tf` repo:
```
$ terraform init
$ terraform apply --var "token=<DO_TOKEN>" --var "ssh_key_id=<KEY_ID>" -var "pvt_key=$HOME/.ssh/id_rsa"
``` 

This can take up to 10mins to complete. Terraform will create 3 Bionic droplets each with a 50GB volume attached. Each droplet will have a private IP address in the `10.10.10.0/24` subnet.

All the resources will be organized in a `ceph-training-tf-<uniqueid>` project.

Note: if you want to destroy/recreate the lab environment, you can run `$ terraform destroy`.

Once terraform is done, setup entries in your local `/etc/hosts` to follow this lab more easily:
```
$ doctl compute droplet list --format PublicIPv4,Name | grep 'ceph-' | sudo tee -a /etc/hosts
144.126.211.15    ceph-1
143.198.70.94     ceph-2
137.184.6.234     ceph-0
$ ssh root@ceph-0 hostname
Warning: Permanently added 'ceph-0' (ED25519) to the list of known hosts.
ceph-0
```
Note: Remember to remove these entries if you re-create the environment.

## Deploying Ceph

In this lab we will deploy Ceph from scratch. A Ceph deployment is always done in the following sequence:
 - Mons
 - Mgrs
 - OSDs

### Deploying the mons

#### First mon deployment

We will deploy one mon on each droplet. The first step is to generate a minimal ceph configuration file:
```
root@ceph-0:~# uuidgen -r
5ad8d39f-794c-406d-93c9-3a30326ee13b

root@ceph-0:~# cat <<EOF > /etc/ceph/ceph.conf
> [global]
> fsid = 5ad8d39f-794c-406d-93c9-3a30326ee13b
> mon_host = 10.10.10.2, 10.10.10.3, 10.10.10.4 #These may differ in your env. double check with: # ip a s dev eth1
> public_network = 10.10.10.0/24
> EOF
```
Copy this configuration file to `ceph-1` and `ceph-2` in `/etc/ceph/ceph.conf`.

Generate a secret (cephx keyring) that will be shared among all mons to form a quorum. The key name is `mon.` (including the `.`):
```
root@ceph-0:~# ceph-authtool --create-keyring /etc/ceph/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
creating /etc/ceph/ceph.mon.keyring

root@ceph-0:~# cat /etc/ceph/ceph.mon.keyring
[mon.]
	key = AQDbehtjpVCMNxAATQGhvaZWjCGd67S/hu5oHA==
	caps mon = "allow *"
```
Copy this keyring file to `ceph-1` and `ceph-2` in `/etc/ceph/ceph.mon.keyring`.

Create the `client.admin` keyring (the equivalent of the cluster root user) and add this key to the `mon.` keyring file on `ceph-0`:
```
root@ceph-0:~# ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap mgr 'allow *' --cap osd 'allow *'
creating /etc/ceph/ceph.client.admin.keyring

root@ceph-0:~# ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
importing contents of /etc/ceph/ceph.client.admin.keyring into /etc/ceph/ceph.mon.keyring
```
Copy `/etc/ceph/ceph.client.admin.keyring` to `ceph-1` and `ceph-2`.

Note: We do not need to copy the updated `/etc/ceph/ceph.mon.keyring` to other hosts. The extra keyring information is going to be used during the creation of the initial quorum (`ceph-0`). `ceph-1` and `ceph-2` will be added into the quorum afterwards.

The next step is to create the initial `monmap`. As mentioned in the previous note, the monmap will contain only `ceph-0` initially to form a quorum of 1 monitor. We will extend the quorum later in the deployment process:
```
root@ceph-0:~# monmaptool --create --add ceph-0 10.10.10.4 --fsid 5ad8d39f-794c-406d-93c9-3a30326ee13b /etc/ceph/monmap
monmaptool: monmap file /etc/ceph/monmap
monmaptool: set fsid to 5ad8d39f-794c-406d-93c9-3a30326ee13b
monmaptool: writing epoch 0 to /etc/ceph/monmap (1 monitors)

root@ceph-0:~# monmaptool --print /etc/ceph/monmap
monmaptool: monmap file /etc/ceph/monmap
epoch 0
fsid 5ad8d39f-794c-406d-93c9-3a30326ee13b
last_changed 2022-09-09T17:51:03.363867+0000
created 2022-09-09T17:51:03.363867+0000
min_mon_release 0 (unknown)
election_strategy: 1
0: v1:10.10.10.4:6789/0 mon.ceph-0
```
IMPORTANT: Make sure that the IP address is correct (`# ip a s dev eth1`) and the the fsid is the same as the one in `/etc/ceph/ceph.conf`

Finally, initialize the monitor store:
```
root@ceph-0:~# chown -R ceph: /etc/ceph/
root@ceph-0:~# ceph-mon --mkfs -i ceph-0 --monmap /etc/ceph/monmap --keyring /etc/ceph/ceph.mon.keyring --setuser ceph --setgroup ceph
root@ceph-0:~# ls /var/lib/ceph/mon/ceph-ceph-0/
keyring  kv_backend  store.db
```

The monitor is deployed and should be able to form a quorum once started:
```
root@ceph-0:~# systemctl enable --now ceph-mon@ceph-0
Created symlink /etc/systemd/system/ceph-mon.target.wants/ceph-mon@ceph-0.service → /lib/systemd/system/ceph-mon@.service.
root@ceph-0:~# ceph -m 10.10.10.4 mon stat
e1: 1 mons at {ceph-0=v1:10.10.10.4:6789/0}, election epoch 3, leader 0 ceph-0, quorum 0 ceph-0
```
Note: the `-m <ip>` option is not required. Since we have 3 mons in our config file (`mon_host`), the ceph cli will randomly try to reach one of them. The `-m` option overrides the configuration file to force querying only the deployed mon.

If the `ceph mon stat` command hangs, there's no quorum.

### Deploying the remaining mons

Now that we have a quorum of one monitor, we are going to add `ceph-1` and `ceph-2` to the quorum. Both should be able to connect to the cluster since we copied the `/etc/ceph/ceph.client.admin.keyring` file locally already:
```
root@ceph-1:~# ceph -m 10.10.10.4 mon stat
e1: 1 mons at {ceph-0=v1:10.10.10.4:6789/0}, election epoch 3, leader 0 ceph-0, quorum 0 ceph-0
root@ceph-2:~# ceph -m 10.10.10.4 mon stat
e1: 1 mons at {ceph-0=v1:10.10.10.4:6789/0}, election epoch 3, leader 0 ceph-0, quorum 0 ceph-0
```

To do so we need each mon to add itself to the latest monmap and initialize their store. Starting with `ceph-1`:
```
root@ceph-1:~# ceph -m 10.10.10.4 mon getmap -o /etc/ceph/monmap
got monmap epoch 1
root@ceph-1:~# monmaptool --add ceph-1 10.10.10.3 /etc/ceph/monmap
monmaptool: monmap file /etc/ceph/monmap
monmaptool: writing epoch 1 to /etc/ceph/monmap (2 monitors)
root@ceph-1:~# chown -R ceph: /etc/ceph/
root@ceph-1:~# ceph-mon --mkfs -i ceph-1 --monmap /etc/ceph/monmap --keyring /etc/ceph/ceph.mon.keyring --setuser ceph --setgroup ceph
root@ceph-1:~# systemctl enable --now ceph-mon@ceph-1
Created symlink /etc/systemd/system/ceph-mon.target.wants/ceph-mon@ceph-1.service → /lib/systemd/system/ceph-mon@.service.
root@ceph-1:~# ceph mon stat
e2: 2 mons at {ceph-0=v1:10.10.10.4:6789/0,ceph-1=[v2:10.10.10.3:3300/0,v1:10.10.10.3:6789/0]}, election epoch 8, leader 0 ceph-0, quorum 0,1 ceph-0,ceph-1
```

Then `ceph-2`:
```
root@ceph-2:~# ceph -m 10.10.10.4 mon getmap -o /etc/ceph/monmap
got monmap epoch 2
root@ceph-2:~# monmaptool --add ceph-2 10.10.10.2 /etc/ceph/monmap
monmaptool: monmap file /etc/ceph/monmap
monmaptool: writing epoch 2 to /etc/ceph/monmap (3 monitors)
root@ceph-2:~# chown -R ceph: /etc/ceph/
root@ceph-2:~# ceph-mon --mkfs -i ceph-2 --monmap /etc/ceph/monmap --keyring /etc/ceph/ceph.mon.keyring --setuser ceph --setgroup ceph
root@ceph-2:~# systemctl enable --now ceph-mon@ceph-2
Created symlink /etc/systemd/system/ceph-mon.target.wants/ceph-mon@ceph-2.service → /lib/systemd/system/ceph-mon@.service.
root@ceph-2:~# ceph mon stat
e3: 3 mons at {ceph-0=v1:10.10.10.4:6789/0,ceph-1=[v2:10.10.10.3:3300/0,v1:10.10.10.3:6789/0],ceph-2=[v2:10.10.10.2:3300/0,v1:10.10.10.2:6789/0]}, election epoch 12, leader 0 ceph-0, quorum 0,1,2 ceph-0,ceph-1,ceph-2
```

Make sure all mons runs the latest version of the messaging protocol:
```
# ceph mon enable-msgr2
 ```

We now have a quorum of 3 monitors and can start deploying the managers. 

### Deploying the mgrs

Deploy one manager on each node. We need to:
 - Create a local directory for the process
 - Create a keyring and store it in the process directory

Let's start with `ceph-0`:
```
root@ceph-0:~# mkdir /var/lib/ceph/mgr/ceph-ceph-0
root@ceph-0:~# ceph auth get-or-create mgr.ceph-0 mon 'allow profile mgr' osd 'allow *' mds 'allow *' -o /var/lib/ceph/mgr/ceph-ceph-0/keyring
root@ceph-0:~# systemctl enable --now ceph-mgr@ceph-0
Created symlink /etc/systemd/system/ceph-mgr.target.wants/ceph-mgr@ceph-0.service → /lib/systemd/system/ceph-mgr@.service.
root@ceph-0:~# ceph mgr stat
{
    "epoch": 4,
    "available": true,
    "active_name": "ceph-0",
    "num_standby": 0
}
```
Note: The `mds` (metadata server) is a cephfs component and not part of this training.

Same for `ceph-1`:
```
root@ceph-1:~# mkdir /var/lib/ceph/mgr/ceph-ceph-1
root@ceph-1:~# ceph auth get-or-create mgr.ceph-1 mon 'allow profile mgr' osd 'allow *' mds 'allow *' -o /var/lib/ceph/mgr/ceph-ceph-1/keyring
root@ceph-1:~# systemctl enable --now ceph-mgr@ceph-1
Created symlink /etc/systemd/system/ceph-mgr.target.wants/ceph-mgr@ceph-1.service → /lib/systemd/system/ceph-mgr@.service.
```

And for `ceph-2`:
```
root@ceph-2:~# mkdir /var/lib/ceph/mgr/ceph-ceph-2
root@ceph-2:~# ceph auth get-or-create mgr.ceph-2 mon 'allow profile mgr' osd 'allow *' mds 'allow *' -o /var/lib/ceph/mgr/ceph-ceph-2/keyring
root@ceph-2:~# systemctl enable --now ceph-mgr@ceph-2
Created symlink /etc/systemd/system/ceph-mgr.target.wants/ceph-mgr@ceph-2.service → /lib/systemd/system/ceph-mgr@.service.
```

We can check that all our mgrs are correctly deployed by looking at the `num_standby` number in `ceph mgr stat`:

```
root@ceph-2:~# ceph mgr stat
{
    "epoch": 15,
    "available": true,
    "active_name": "ceph-0",
    "num_standby": 2
}
```

We have 3 mgrs deployed (one active, two standby). We can start deploying the OSDs.

### Deploying the OSDs

To deploy the OSDs we will use the `ceph-volume` command. The only local file needed is a bootstrap keyring that gives a minimal amount of permission to deploy an OSD.

```
root@ceph-0:~# ceph auth get client.bootstrap-osd -o /var/lib/ceph/bootstrap-osd/ceph.keyring
exported keyring for client.bootstrap-osd
root@ceph-0:~# lsblk /dev/sda
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda    8:0    0  50G  0 disk
root@ceph-0:~# ceph-volume lvm prepare --data /dev/sda
[...]]
 --> ceph-volume lvm prepare successful for: /dev/sda

root@ceph-0:~#  ceph-volume lvm activate --all
[...]
--> ceph-volume lvm activate successful for osd ID: 0
```

Repeat for `ceph-1` and `ceph-2`. Once done check that all OSDs have been deployed:
```
root@ceph-0:~# ceph osd stat
3 osds: 3 up (since 41s), 3 in (since 55s); epoch: e25
```

That's all for the lab. Make sure to check the lab video for more details.
