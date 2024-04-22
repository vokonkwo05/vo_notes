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
```$ ipmitool -U root -I lanplus -H BMC_IP -P $PASS chassis power diag```

Reset BMC:
``` # ipmitool mc reset cold```

Check temp:
```# ipmitool sdr elist | grep -i inl```



## Repaves

Check repave status: 
``` # st2 run digitalocean.hms hosts=$SERVER```

Repave server:
``` # st2 run --async digitalocean.provision hosts=$SERVER hpw_workflow_wait=false release=true```


## Sunset

Sunset server: 
```st2 run digitalocean.sunset_workflow hosts=nyc3node4148,nyc3node4135 tower_username="${LDAP_USERNAME}" tower_password="${LDAP_PASSWORD}" jira_ticket=OPS-34167 --async```


If secure erase is not needed, you can remove from chef and delete from alpha:




## Migrations

Migrate the top 10 droplets: 
```/usr/local/bin/migrate droplet --safe $(ps -aux  --sort=-%cpu | grep qemu-system-x86_64 | grep Droplet- | cut -d = -f 2  | cut -d , -f 1 | cut -d - -f 2 | head -n 10 | xargs)```

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
