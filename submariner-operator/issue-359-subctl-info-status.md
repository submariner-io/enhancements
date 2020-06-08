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


* _subctl status gateways_
    * endpoint:  
        backend: strongswan  
        cable_name: submariner-cable-cluster3-172-17-0-10  
        cluster_id: cluster3  
        hostname: cluster3-worker  
        nat_enabled: false  
        private_ip: 172.17.0.10  
        public_ip: ""  
        subnets:  
        - 100.93.0.0/16  
        - 10.243.0.0/16  
        status: connected  
    * localEndpoint:  
      backend: strongswan  
      cable_name: submariner-cable-cluster2-172-17-0-5  
      cluster_id: cluster2  
      hostname: cluster2-worker  
      nat_enabled: false  
      private_ip: 172.17.0.5  
      public_ip: ""  
      subnets:  
      - 100.92.0.0/16  
      - 10.242.0.0/16  
      status: connecting  
      statusMessage: Connecting to 5.144.50.234:500  

* _subctl status daemonset_  
    * routeAgentDaemonSetStatus:  
      currentNumberScheduled: 2  
      desiredNumberScheduled: 2  
      numberAvailable: 2  
      numberMisscheduled: 0  
      numberReady: 2  
      observedGeneration: 1  
      updatedNumberScheduled: 2  
    * engineDaemonSetStatus:  
      currentNumberScheduled: 1  
      desiredNumberScheduled: 1  
      numberAvailable: 1  
      numberMisscheduled: 0  
      numberReady: 1  
      observedGeneration: 1  
      updatedNumberScheduled: 1  

* _subctl info_  
   * Discovered network details:  
         Network plugin:  weave-net  
         Service CIDRs: [100.92.0.0/16]  
         Cluster CIDRs: [10.242.0.0/16]    
   * endpoint:  
        backend: strongswan  
        cable_name: submariner-cable-cluster3-172-17-0-10  
        cluster_id: cluster3  
        hostname: cluster3-worker  
        nat_enabled: false  
        private_ip: 172.17.0.10  
        public_ip: ""  
        subnets:  
        - 100.93.0.0/16  
        - 10.243.0.0/16  
      status: connected  
   * localEndpoint:  
      backend: strongswan  
      cable_name: submariner-cable-cluster2-172-17-0-5  
      cluster_id: cluster2  
      hostname: cluster2-worker  
      nat_enabled: false  
      private_ip: 172.17.0.5  
      public_ip: ""  
      subnets:  
      - 100.92.0.0/16  
      - 10.242.0.0/16  
      status: connecting  
      statusMessage: Connecting to 5.144.50.234:500  
   * routeAgentDaemonSetStatus:  
            currentNumberScheduled: 2  
            desiredNumberScheduled: 2  
            numberAvailable: 2  
            numberMisscheduled: 0  
            numberReady: 2  
            observedGeneration: 1  
            updatedNumberScheduled: 2  
   * engineDaemonSetStatus:  
            currentNumberScheduled: 1  
            desiredNumberScheduled: 1  
            numberAvailable: 1  
            numberMisscheduled: 0  
            numberReady: 1  
            observedGeneration: 1  
            updatedNumberScheduled: 1  
      


