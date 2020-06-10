#Issue 359: Add more information to subctl info cmd

## Summary
Current cmd:  
* subctl info  
    Discovered network details:  
         Network plugin:  weave-net  
         Service CIDRs: [100.92.0.0/16]  
         Cluster CIDRs: [10.242.0.0/16]


## Motivation
To supply more information on current deployment status

## Proposal
* _subctl status network_  
    Discovered network details:  
         Network plugin:  weave-net  
         Service CIDRs: [100.92.0.0/16]  
         Cluster CIDRs: [10.242.0.0/16]  
         GlobalCIDR:    [<>]



* _subctl status gateways_  
  NODE       HA-STATUS       CONNECTIONS    SUMMARY  
  worker1    active          3/3            All connections OK  
  worker2    passive         0/0                                
  
    
* _subctl status connections_  
  GATEWAY  CLUSTER    REMOTE-IP    DRIVER     SUBNETS                         STATUS  
  worker1  cluster2   172.17.0.10  libreswan  100.93.0.0/16,  10.243.0.0/16   connected  
  worker1  cluster3   172.17.0.11  libreswan  100.94.0.0/16,  10.244.0.0/16   connected  
  
  
* _subctl status endpoints_
  * Cable driver: strongswan  
  * Cluster ID: cluster3  
  * Endpoint IP: <public IP> (172.17.0.10)  


* _subctl info_  
   * Discovered network details:  
         Network plugin:  weave-net  
         Service CIDRs: [100.92.0.0/16]  
         Cluster CIDRs: [10.242.0.0/16]    
         GlobalCIDR:    [<>]  
   * Service status: all ok / there is an issue with route agents (3/5 correctly scheduled)  
   * Connections: All OK (3/3)
   
      


