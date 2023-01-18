---
Title: Static VXLAN tunnels
---

In VXLAN-based networks, there are a range of complexities and challenges in determining the destination virtual tunnel endpoints (VTEPs) for any given VXLAN. At scale, solutions such EVPN try to address these complexities, however, they also have their own complexities.

Static VXLAN tunnels serve to connect two VTEPs in a given environment.
Static VXLAN tunnels are the simplest deployment mechanism for small scale environments and are interoperable with other vendors that adhere to VXLAN standards. Because you simply map which VTEPs are in a particular VNI, you can avoid the tedious process of defining connections to every VLAN on every other VTEP on every other rack.

Static Virtual Extensible LAN (VXLAN), also known as unicast VXLAN, enables you to statically configure source and destination virtual tunnel endpoints (VTEPs) for a particular traffic flow (VNI).

A source VTEP encapsulates and a destination VTEP de-encapsulates Layer 2 packets with a VXLAN header, thereby tunneling the packets through an underlying Layer 3 IP network.

## Benefits of Static VXLAN

Instead of using an Ethernet VPN (EVPN) control plane to learn the MAC addresses of hosts, static VXLAN uses a flooding-and-learning technique in the VXLAN data plane.

Therefore, using static VXLAN reduces complexity in the control plane.
Static VXLAN provides the benefits of VXLAN and is relatively easy to design and configure. 

## How Static VXLAN Works

To enable static VXLAN on a device that functions as a VTEP, you must configure:

- A list that includes one or more remote VTEPs with which the local VTEP can form a VXLAN tunnel.
- Ingress node replication.
- The VTEP’s loopback interface (lo0):
  - Configure an anycast IP address as the primary interface.
  - Specify this interface as the source interface for a VXLAN tunnel. 

When a VTEP receives a broadcast, unknown unicast, or multicast (BUM) packet, the VTEP uses ingress node replication to replicate and flood the packet to the statically defined remote VTEPs on your list.

The remote VTEPs in turn flood the packet to the hosts in each VXLAN of which the VTEPs are aware.

The VTEPs learn the MAC addresses of remote hosts from the VTEPs on the remote VTEP list and the MAC addresses of local hosts from the local access interfaces.

Upon learning a MAC address of a host, the MAC address is then added to the Ethernet switching table.

In this environment, static VXLAN serves two purposes:
- To learn the MAC addresses of hosts in a VXLAN. To accomplish this task, static VXLAN uses the ingress node replication feature to flood broadcast, unknown unicast, and multicast (BUM) packets throughout a VXLAN. The VTEPs learn the MAC addresses of remote hosts from the VTEPs on the remote VTEP list and the MAC addresses of local hosts from the local access interfaces. Upon learning of a host’s MAC address, the MAC address is then added to the Ethernet switching table.
- To encapsulate Layer 2 packets with a VXLAN header and later de-encapsulate the packets, thereby enabling them to be tunneled through an underlying Layer 3 IP network. For this task to be accomplished, on each VTEP, you configure a list of statically defined remote VTEPs with which the local VTEP can form a VXLAN tunnel.

## Requirements

For a basic VXLAN configuration, make sure that:
- The VXLAN has a network identifier (VNI).
- Bridge learning must be enabled on the VNI (bridge learning is disabled by default).
- The VXLAN link and local interfaces are added to the bridge to create the association between the port, VLAN and VXLAN instance.
- Each traditional mode bridge on the switch has only one VXLAN interface.

## Configure Static VXLAN Tunnels

To configure static VXLAN tunnels, do the following on each leaf:
- Specify an IP address for the loopback.
- Create a VXLAN interface using the loopback address for the local tunnel IP address.
- Enable bridge learning on the VNI.
- Create the tunnels by configuring the remote IP address to each other leaf switch’s loopback address.

---

Source:
- https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-44/Layer-2/Ethernet-Bridging-VLANs
- https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-43/Network-Virtualization/Static-VXLAN-Tunnels/
- https://www.juniper.net/documentation/us/en/software/junos/evpn-vxlan/topics/topic-map/vxlan-static.html
- https://www.redhat.com/sysadmin/kubernetes-pods-communicate-nodes
