## Add Built-in Benchmarking Tool

https://github.com/submariner-io/submariner-website/issues/40

## Summary

This is a proposal to support a built-in benchmarking tool in Submariner.
The goal is to provide users with easy way to benchmark the network performance across clusters, and experiment with different tuning of the system.


## Proposal

Add a new ability to have a built-in benchmark tool.
Implement a new Subctl command that will use iperf/netperf internally to benchmark the throughput and round trip latency between two nodes.

Scheduling server and client can be placed within the same cluster or spread across multiple clusters.
The user can specify source and destination nodes via node labels.

If node labels are not specified, the following tests will be run by default:
* non-gateway to non-gateway  
* gateway to gateway

In case server-options are not included, the command will use defaults.

*Note* - In phase 1 of the implementations there will be no additional params other then `<kubeconfing1>` and `<kubeconfig2>`

#### Backend tools and params

The backend for latency test will be netperf.   
The default params that will be passed to netperf:
```
-t TCP_RR  -- -o min_latency,mean_latency,max_latency,stddev_latency,transaction_rate
```
* -t: testname. TCP_RR - user-space to user-space ping with no connection time (request/response test)  
* -o: sets the style of output to “CSV” where there will be one line of comma-separated values, preceded by one line of column names  

The backend for throughput tests will be iperf. The default params that will be passed to iperf:
```
-w 8M -P 10 -c
```
* -w: TCP window size `n[KM]`  
* -P: The number of simultaneous connections to make to the server    
* -c: client mode    

#### Source code location

Source code should go into Submariner.   
There is going to be a dependency on the nettest image from Shipyard, which will need to have the necessary tools.


### Subctl command

```
subctl test throughput <kubeconfig1> <kubeconfig2>
subctl test latency <kubeconfig1> <kubeconfig2>
```
