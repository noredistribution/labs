## Testing EVPN IRB with cEOS-lab

### Topology

PC1 (et1) --- (et3) Leaf1 (et1) --- (et1) Spine1 (et2) --- (et1) Leaf2 (et3) --- (et1) PC2

### Testbed

Server: Docker running on Centos 7.7

Container image: Arista cEOS-lab 4.24.2.1F

### Deployment

I've used [https://github.com/networkop/docker-topo](https://github.com/networkop/docker-topo) to build the topology which eases 
docker network topology creations a lot!

Works both with and without virtualenv (python3 is required).

```
python3 -m virtualenv -p (which python3) venv; cd venv
source bin/activate
pip install git+https://github.com/networkop/docker-topo.git
```

Copy the yml file and the config dir

`cp -r ~/projects/labs/ceos-lab-evpn-irb/* topo-extra-files/examples/v2/`

Create the topology

```
cd topo-extra-files/examples/v2/
docker-topo --create evpn.yml
```

If all went well you should see something like this:

```
INFO:__main__:
alias Spine-1='docker exec -it irb_Spine-1 Cli'
alias Leaf-1='docker exec -it irb_Leaf-1 Cli'
alias Leaf-2='docker exec -it irb_Leaf-2 Cli'
alias ceosPC1='docker exec -it irb_ceosPC1 Cli'
alias ceosPC2='docker exec -it irb_ceosPC2 Cli'
INFO:__main__:All devices started successfully
```

Now you can exec into all of the containers. To make end-to-end connectivity work in this cEOS-lab version Linux RPF check has to be disabled 
for all internal VLANs used by EVPN

Check the EVPN internal vlans

```
# docker exec -it irb_Leaf-1 Cli
Leaf1>en
Leaf1#show vlan dynamic | grep evpn
evpn                      4094
```

List the RPF values (note that you'll have to switch to the correct namespace)

```
Leaf1(config-if-Et3)#cli vrf tenant-blue
Leaf1(vrf:tenant-blue)(config-if-Et3)#bash

Arista Networks EOS shell

bash-4.2# sysctl -a | grep vlan4094.rp_filter
net.ipv4.conf.vlan4094.rp_filter = 1
```

Disable RPF for each EVPN internal vlan (do this on all VTEPs)

```
bash-4.2# echo "0" > /proc/sys/net/ipv4/conf/vlan4094/rp_filter
bash-4.2# echo "0" > /proc/sys/net/ipv4/conf/all/rp_filter
```

Verify that RPF is reset

```
bash-4.2# sysctl -a | grep vlan4094.rp_filter
net.ipv4.conf.vlan4094.rp_filter = 0
```

Test connectivity

```
# docker exec -it irb_ceosPC1 Cli
PC1>en
PC1#ping 10.0.50.11 repeat 1
PING 10.0.50.11 (10.0.50.11) 72(100) bytes of data.
80 bytes from 10.0.50.11: icmp_seq=1 ttl=64 time=23.8 ms

--- 10.0.50.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 23.863/23.863/23.863/0.000 ms
PC1#ping 10.0.60.11 repeat 1
PING 10.0.60.11 (10.0.60.11) 72(100) bytes of data.
80 bytes from 10.0.60.11: icmp_seq=1 ttl=64 time=19.2 ms

--- 10.0.60.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 19.241/19.241/19.241/0.000 ms
```

Useful commands:

```
Leaf1#show bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 123.1.1.3, local AS number 65001
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >     RD: 123.1.1.3:12345 ip-prefix 10.0.40.0/24
                                 -                     -       -       0       i
 * >     RD: 123.1.1.3:12345 ip-prefix 10.0.45.0/24
                                 -                     -       -       0       i
 * >     RD: 123.1.1.5:12345 ip-prefix 10.0.50.0/24
                                 2.2.2.2               -       100     0       65000 65002 i
 * >     RD: 123.1.1.5:12345 ip-prefix 10.0.55.0/24
                                 2.2.2.2               -       100     0       65000 65002 i
 * >     RD: 123.1.1.3:12345 ip-prefix 10.0.60.0/24
                                 -                     -       -       0       i
 * >     RD: 123.1.1.5:12345 ip-prefix 10.0.60.0/24
                                 2.2.2.2               -       100     0       65000 65002 i
 * >     RD: 123.1.1.3:12345 ip-prefix 223.255.255.0/24
   

                              -                     -       -       0       i
Leaf1#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 123.1.1.3, local AS number 65001
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >     RD: 123.1.1.5:60 mac-ip 1000.d25e.1630
                                 2.2.2.2               -       100     0       65000 65002 i
 * >     RD: 123.1.1.5:60 mac-ip 1000.d25e.1630 10.0.60.11
                                 2.2.2.2               -       100     0       65000 65002 i
 * >     RD: 123.1.1.3:60 mac-ip 1052.0ac5.c609
                                 -                     -       -       0       i
 * >     RD: 123.1.1.3:60 mac-ip 1052.0ac5.c609 10.0.60.10


                                 -                     -       -       0       i
Leaf1#show int vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback1 and is active with 1.1.1.1
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is
    [40, 40]          [45, 45]          [60, 60]
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4094, 12345]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is
   [tenant-blue, 12345]
  Headend replication flood vtep list is:
    60 2.2.2.2
  MLAG Shared Router MAC is 0000.0000.0000
```

