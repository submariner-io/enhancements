## Add more capabilities to check status from subctl

https://github.com/submariner-io/submariner-operator/issues/359


## Summary
The current info command is very limited and does not provide any information on current deployment status, only the discovered network details.
```bash 
subctl info  
  Discovered network details:  
        Network plugin:  weave-net  
        Service CIDRs: [100.92.0.0/16]  
        Cluster CIDRs: [10.242.0.0/16]
```

## Motivation
To supply more information on current deployment status

## Proposal
```bash
subctl show networks  
    Discovered network details:  
         Network plugin:  weave-net  
         Service CIDRs: [100.92.0.0/16]  
         Cluster CIDRs: [10.242.0.0/16]  
         GlobalCIDR:    [<>] 
```



``` bash 
subctl show gateways  
NODE       HA-STATUS       CONNECTIONS    SUMMARY  
worker1    active          3/3            All connections OK  
worker2    passive         0/0                                
```  
    
```bash
subctl show connections  
GATEWAY  CLUSTER    REMOTE-IP    DRIVER     SUBNETS                         STATUS  
worker1  cluster2   172.17.0.10  libreswan  100.93.0.0/16,  10.243.0.0/16   connected  
worker1  cluster3   172.17.0.11  libreswan  100.94.0.0/16,  10.244.0.0/16   connected
```
  
  
```bash 
subctl show endpoints  
  Cable driver: strongswan  
  Cluster ID: cluster3  
  Endpoint IP: <public IP> (172.17.0.10) 
  ```


```bash
subctl show all  
Discovered network details:  
  Network plugin:  weave-net  
  Service CIDRs: [100.92.0.0/16]  
  Cluster CIDRs: [10.242.0.0/16]    
  GlobalCIDR:    [<>]  

Cable driver: strongswan  
Cluster ID: cluster3  
Endpoint IP: <public IP> (172.17.0.10) 

NODE       HA-STATUS       CONNECTIONS    SUMMARY  
worker1    active          3/3            All connections OK  
worker2    passive         0/0                                

GATEWAY  CLUSTER    REMOTE-IP    DRIVER     SUBNETS                         STATUS  
worker1  cluster2   172.17.0.10  libreswan  100.93.0.0/16,  10.243.0.0/16   connected  
worker1  cluster3   172.17.0.11  libreswan  100.94.0.0/16,  10.244.0.0/16   connected

COMPONENT VERSION
submariner-operator v0.4.0
submariner-engine v0.4.0
submariner-routeagent v0.4.0
submariner-globalnet v0.4.0
lighthouse-agent v0.4.0
   ```
   
      
```bash   
subctl show versions 
COMPONENT VERSION
submariner-operator v0.4.0
submariner-engine v0.4.0
submariner-routeagent v0.4.0
submariner-globalnet v0.4.0
lighthouse-agent v0.4.0
```


