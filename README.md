# Azure network performance test
This template provision environment for network performance testing and installs qperf as daemon for latency tests and iperf as daemon for troughput tests.

Testing scenarios:
* Latency and performance between VMs in the same proxymity placement group
* Latency and performance between VMs in the same availability zone
* Latency and performance between VMs across availability zones
* Latency and performance between VMs across regions

VMs deployed:
* r1-z1-pg1-vm01
* r1-z1-pg1-vm02
* r1-z1-vm
* r1-z2-vm
* r2-vm

Deployment notes:
* username is net
* password is provided as parameter
* primary and secondary region is provided as parameter
* by default full topology is created, but you can use boolean parameters to disable some tests

To deploy via CLI:
```bash
az group create -n netperf -l westeurope
az group deployment create -g netperf \
    --template-file deploy.json \
    --parameters password=Azure12345678 \
    --parameters primaryRegion=westeurope \
    --parameters secondaryRegion=northeurope
```

To deploy via portal:

[![Deploy to Azure](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/deploytoazure.png)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Ftkubica12%2Fazure-network-performance-test%2Fmaster%2Fazuredeploy.json){:target="_blank"}

# Test network performance
Connect to r1-z1-pg1-vm01
```bash
export ip=$(az network public-ip show -n r1-z1-pg1-vm01-ip -g netperf --query ipAddress -o tsv)
ssh net@$ip
```

Test bandwidth to other VMs
```bash
sudo iperf -c r1-z1-pg1-vm02 -i 1 -t 30 # same proximity placement group
sudo iperf -c r1-z1-vm -i 1 -t 30 # same availability zone
sudo iperf -c r1-z2-vm -i 1 -t 30 # different availability zone
sudo iperf -c 10.1.0.4 -i 1 -t 30 # secondary region via VNET peering
sudo iperf -c 10.1.0.4 -P16 -t 30 # multiple parallel sessions
```

Test latency to vm2 in the same zone and vm3 in different zone.
```bash
qperf -t 20 -v r1-z1-pg1-vm02 tcp_lat # same proximity placement group
qperf -t 20 -v r1-z1-vm tcp_lat # same availability zone
qperf -t 20 -v r1-z2-vm tcp_lat # different availability zone
qperf -t 20 -v 10.1.0.4 tcp_lat # secondary region via VNET peering
```

What we expect to happen and what to do?
* Network troughput
  * Expect troughtput close to specs in documentation (8 Gbps for Standard_D16s_v3)
  * Troughtput will be very stable for traffic inside proximity group and inside zone
  * Troughtput will be almost the same between different zones, but might fluctulate a little from time to time, but typically less than 1%
  * Throughput between regions is WAN link so expected to be slower. Typical behavior is that with single connection speed is very good, but significantly slower than inside region, but using multiple sessions drives performance closer to what is seen inside regions (but might fluctulate a more)
* Latency
  * Thanks to accelerated networking available on some machines including Standard_D16_v3 latency is very good
  * Latency inside proximity placement group will be very stable, low tens of microseconds. Proximty placement group guarantees close proximity of VMs.
  * Latency within zone will be also extremly low and might be the same as within placement group, because VM can be scheduled also close to first one, but it is not guaranteed. But even then latency is just couple of tens of microseconds.
  * Latency between zones is likely to be 20% higher then within zone, but still way bellow 100 microseconds so very well suitable for synchronous operations
  * Latency between regions is good for WAN link, but obviously suited more for async operations. Eg. between West Europe and North Europe you will see around 10 miliseconds latency. 
  
**Do not use ping to test latency** - it has very low priority in Azure network as well as OS TCP/IP stack
**Do not use file copy to test network throughput** as storage can be most limiting factor

Please note proximity placement groups can limit your scaling options and VM choices, refer to [documentation](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/co-location#best-practices)