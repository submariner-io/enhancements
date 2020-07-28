## Add Built-in Benchmarking Tool

## Summary

There is a need to support a built-in benchmarking tool in Submariner.

## Proposal

Add a new ability to have a built-in benchmark tool.
Implement a new Subctl command that will use iperf3 internally to benchmark the throughput between the two nodes.

The feature Should support scheduling server and client in the same cluster.

The user can specify source and destination nodes via node names. (Or via lables?)

If nodes are not specified, the following will be the default tests: 
* non-gateway to non-gateway  
* gateway to gateway


#### Source code location

Source code should go into Submariner. 
There is going to be a dependency on Shipyard (iperf image)


### Subctl command

```
subctl test throughput --server-config, --server-node, --client-config and --client-node
```
