## Add Built-in Benchmarking Tool

https://github.com/submariner-io/submariner-website/issues/40

## Summary

This is a proposal to support a built-in benchmarking tool in Submariner.
The goal is to provide users with easy way to benchmark the network performance across clusters, and experiment with different tuning of the system.


## Proposal

Add a new ability to have a built-in benchmark tool.
Implement a new Subctl command that will use uperf internally to benchmark the throughput and round trip latency between two nodes.

Scheduling server and client can be placed within the same cluster or spread across multiple clusters.

The user can specify source and destination nodes via node labels.

If node labels are not specified, the following tests will be run by default:
* non-gateway to non-gateway  
* gateway to gateway

In case server-options are not included, the command will use defaults.

The backend for latency test will be netperf. The default params that will be passed to netperf:
```
-t TCP_RR  -- -o min_latency,mean_latency,max_latency,stddev_latency,transaction_rate
```

The backend for throughput will be iperf. The default params that will be passed to iperf:
```
-w 8M -P 10 -c
```

#### Source code location

Source code should go into Submariner. 
There is going to be a dependency on the nettest image from Shipyard (Which has iperf).


### Subctl command

```
subctl test throughput <kubeconfig1> <kubecinfig2> --node-label --server-options
subctl test latency <kubeconfig1> <kubecinfig2> --node-label --server-options
```
