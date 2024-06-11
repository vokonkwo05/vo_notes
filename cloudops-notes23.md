# CloudOps Notes 

## hypervisor

Get hypervisor info: 
``` $ hypervisor info $SERVER```

Show all running and queued events:
```# /opt/apps/hvdclient/bin/hvdclient hv eventinfo -s RUNNING -s QUEUED```

show event info:
```# hvdclient hv eventinfo -t $eventID```

zombiecheck:
```# zombiecheck 'hostname -s'```

clean hypervisors:
```zombiecheck -c `hostname -s'```

[Restart hypervisor networking](https://github.com/digitalocean/documentation/blob/master/oncall/playbooks/procedures/restart-hypervisor-networking.md) Deletes and recreates droplet flows
```# /sbin/net-core --recover```


Check for DDOS
```tcpdump -nnn -i bond0 -c 5000 | awk '{print $5}' | cut -f 1,2,3,4 -d '.' | sort | uniq -c | sort -nr | head -n 20```


Kdump:
This command is run in a jump server:
```$ ipmitool -I lanplus -U root -H $BMC_1P -P $IPMI_PASS chassis power diag```

power_cycle a node: ```$ ipmitool -I lanplus -U root -H $BMC_1P -P $IPMI_PASS chassis power cycle```

Reset BMC:
``` # ipmitool mc reset cold```

Check temp:
```# ipmitool sdr elist | grep -i inl```



## Repaves

Check repave status: 
``` # st2 run digitalocean.hms hosts=$SERVER```

Repave server:
``` # st2 run --async digitalocean.provision hosts=$SERVER hpw_workflow_wait=false release=true train=test```


## Sunset

Sunset server: 
```st2 run digitalocean.sunset_workflow hosts=nyc3node4148,nyc3node4135 tower_username="${LDAP_USERNAME}" tower_password="${LDAP_PASSWORD}" jira_ticket=OPS-34167 --async```


If secure erase is not needed, you can remove from chef and delete from alpha:




## Migrations

Migrate the top 10 droplets: 
```/usr/local/bin/migrate droplet --safe --no-confirm $(ps -aux  --sort=-%cpu | grep qemu-system-x86_64 | grep Droplet- | cut -d = -f 2  | cut -d , -f 1 | cut -d - -f 2 | head -n 10 | xargs)```

Check region stats: 
``` # migrate statistics nyc3```




## NAS 
[NAS notes](https://do-internal.atlassian.net/wiki/spaces/CO/pages/536969368/NAS+procedures)

Get NAS status: 
```# /opt/apps/imagemanagement/bin/imagectl --region $REGION backend get --name $NAS```

Put NAS in `readonly` status:
```# /opt/apps/imagemanagement/bin/imagectl --region <region> backend set-inventory-status --name <nas> --status readonly```




# CEPH

SSH to monitor node: 
```# ssh fra1b2```

Check for down OSDs:
```# ceph osd tree down```

Find the host where the OSD resides: 
```ceph osd find $osdID```

list OSDs:
```# /opt/apps/storman/bin/storman list | grep $OSD```



## Kubernetes

List  nodes in the cluster: 
```# kubectl --kubeconfig=kubeconfig-file get nodes```

Get CPC config info: 
```kubectl --kubeconfig=kubeconfig-cpc-92 get pods -n $clusterID```


Get application logs:
```kubectl --kubeconfig=kubeconfig-$CLUSTERNAME logs -n namespace podname```

Get events: 
```kubectl --kubeconfig=kubeconfig-$CLUSTERNAME get events -n kube-system```

Describe pod:

```kubectl --kubeconfig=kubeconfig-$CLUSTERNAME describe pod -n namespace  pod```

## Droplet migration

When a high_load alert, run coctl in the docker container to see if there is any abuse

Check for abuse:
```coctl leecher $SERVER```

If there is no abuse, migrate (while ssh'ed in the HV) top 10 CPU usage Droplets using this command:
```/usr/local/bin/migrate droplet --safe --no-confirm $(ps -aux --sort=-%cpu | grep qemu-system-x86_64 | grep Droplet- | cut -d = -f 2 | cut -d , -f 1 | cut -d - -f 2 | head -n 10 | xargs)```

## More on migration

Migrate all powered down Droplets:
```/usr/local/bin/migrate droplet $(ls -lsh /var/lib/libvirt/images | grep "root" | grep "raw"  | awk '{print $10}' | cut -d. -f1 | xargs)```

Migrate five of the largest disk images on the HV:
```/usr/local/bin/migrate droplet $(ls -lsh /var/lib/libvirt/images | sort -nrk1 | grep "raw" | grep -v "config" | head -5 | awk '{print $10}' |cut -d. -f1 | xargs)```

Migrate droplets with an existing OPS ticket:
```migrate evac $SERVER --emergency --fallback --jira $OPS-TICKET```

### Delete a droplet from a hypervisor

Help page:
```droplet-admin -h```

Example usage:
```droplet-admin archive -f -e vokonkwo@digitalocean.com $droplet_id```

### Remove a seaworthy droplet caused by a repave attempt

verify droplet is running:
```sudo virsh list --all```

Destroy droplet:
```sudo virsh destroy Droplet-<droplet_id>```

Undefine droplet:
```sudo virsh undefine Droplet-<droplet_id>```

Change droplet status to archive:
```droplet-admin archive --email=DO_email@digitalocean.com <droplet_id> --force```

### Provision a new storage node (run these commands after a PDU nap)

first:
```source env/production blr1```

then:
```st2 run digitalocean.provision hosts=SGH411NNN2 role=infra-storage-block-osd host_name=prod-data11-block03.blr1.internal.digitalocean.com train=test hpw_workflow_wait=false --async```

### Commands to find errors in a server

Run these commands when ssh'ed into the HV:

```sudo dmesg -T --level=err```

```sudo ipmitool sel elist | egrep $(date +‘%m/%d/%Y’)```

```sudo dmesg -T|egrep 'err|crit|fail|raid'```

Run as root for these:
```cd /var/log && cat kern.log | grep error```

### To remove files from a particular date
```sudo find $Path_to_file -type f -name "*.gz" -newermt 2024-05-27 ! -newermt 2024-05-28 -exec rm {} \;```

## Regex

Comma seperated to space seperated:
```echo "sfo3node905,sfo3node918,sfo3node912,sfo3node906,sfo3node916" | sed 's/,/ /g'```

## Safety checks

Confirm evac is completed:
```L04P999PGY:ansible-playbooks vokonkwo$ ansible-playbook -i sfo3node905, check_evac_status.yaml```

## OSD redeploy

ssh into boxfrom jump; example: ```ssh prod-data08-object01.fra1.internal.digitalocean.com```

Run dmesg to see where DCOPS attached disk: ```dmesg -T```

ssh into the box (through jump) and become root, then run the command:
```/opt/apps/storman/bin/storman deploy --osd=$OSD_NUMBER_ON_PG_ALERT --disk $PARTITION_WHERE_DISK_WAS_MOUNTED_BY_DCOPS```

## network_speed

Check for cable flapping/down: ```sudo dmesg | grep eth```

## seaworthy failures

If a machine fails seaworthy with ```not ok - Run l3mpls-check-bird-sessions```, restart bird.service ```sudo systemctl restart bird.service```, then run ```sudo /usr/lib/l3mpls/checks/l3mpls-check-bird-sessions```, and then ```sudo l3mpls-diagnostics -s```