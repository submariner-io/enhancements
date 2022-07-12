# Add more capabilities to check status from subctl

[Submariner-operator#359](https://github.com/submariner-io/submariner-operator/issues/359)

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

```console
$ subctl show networks
    Discovered network details:
         Network plugin:  weave-net
         Service CIDRs: [100.92.0.0/16]
         Cluster CIDRs: [10.242.0.0/16]
         GlobalCIDR:    [<>]
```

```console
$ subctl show gateways
NODE       HA-STATUS       CONNECTIONS    SUMMARY
worker1    active          3/3            All connections OK
worker2    passive         0/0
```

```console
subctl show connections
GATEWAY  CLUSTER    REMOTE-IP    CABLE-DRIVER     SUBNETS                         STATUS
worker1  cluster2   172.17.0.10  libreswan        100.93.0.0/16,  10.243.0.0/16   connected
worker1  cluster3   172.17.0.11  libreswan        100.94.0.0/16,  10.244.0.0/16   connected
```

```console
subctl show endpoints
CLUSTER-ID    ENDPOINT-IP     PUBLIC-IP     CABLE-DRIVER  TYPE
cluster3      172.17.0.10                   libreswan     local
cluster2      172.17.0.5                    libreswan     remote
```

```console
$ subctl show versions
COMPONENT             VERSION
submariner-operator   0.4.0
submariner            0.4.0
service-discovery     0.4.0
```

```console
$ subctl show all
Discovered network details:
  Network plugin:  weave-net
  Service CIDRs: [100.92.0.0/16]
  Cluster CIDRs: [10.242.0.0/16]
  GlobalCIDR:    [<>]

CLUSTER-ID    ENDPOINT-IP   PUBLIC-IP     CABLE-DRIVER  TYPE
cluster3      172.17.0.10                 libreswan     local
cluster2      172.17.0.5                  libreswan     remote

NODE       HA-STATUS       CONNECTIONS    SUMMARY  
worker1    active          3/3            All connections OK
worker2    passive         0/0

GATEWAY  CLUSTER    REMOTE-IP    CABLE-DRIVER     SUBNETS                         STATUS
worker1  cluster2   172.17.0.10  libreswan        100.93.0.0/16,  10.243.0.0/16   connected
worker1  cluster3   172.17.0.11  libreswan        100.94.0.0/16,  10.244.0.0/16   connected

COMPONENT             VERSION
submariner-operator   0.4.0
submariner            0.4.0
service-discovery     0.4.0
```
