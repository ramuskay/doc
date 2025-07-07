# Baremetal install

The install is based on Flatcar on 3 mini PC  
For flatcar installation we are relying on the following components :

- PXE
- DHCP
- Butane+Ignition
- Raspberry PI as a controller (HTTP server)

Here a diagram on how things works :

```mermaid
sequenceDiagram
    participant Client as iPXE Client (Booting Machine)
    participant DHCP_Server as DHCP Server
    participant HTTP_Server as HTTP Server

    Note over Client,DHCP_Server: iPXE client boots and broadcasts DHCPDISCOVER
    Client->>DHCP_Server: DHCPDISCOVER + iPXE options (requests boot info)
    DHCP_Server-->>Client: DHCPOFFER + IP address + boot file URL (HTTP)
    Client->>DHCP_Server: DHCPREQUEST (accept offer)
    DHCP_Server-->>Client: DHCPACK (confirm IP & boot info)

    Note over Client,HTTP_Server: iPXE client downloads boot script via HTTP
    Client->>HTTP_Server: HTTP GET boot script boot.ipxe
    HTTP_Server-->>Client: boot script

    Note over Client,HTTP_Server: Download images via HTTP
    Client->>HTTP_Server: HTTP GET flatcar_production_pxe.vmlinuz + image.cpio.gz
    HTTP_Server-->>Client: Images downloaded

    Note over Client,HTTP_Server: First Ignition of the server to install image on disk
    Client->>HTTP_Server: HTTP GET <mac_address>_preinstall.ign
    HTTP_Server-->>Client: Ignition started

    Client->>Client: Reboot

    Note over Client,HTTP_Server: Second Ignition of the server to do the postinstall
    Client->>HTTP_Server: HTTP GET <mac_address>_postinstall.ign
    HTTP_Server-->>Client: Ignition started
```

3 features are available to help building around this cluster, all these features are shell scripts :

- `generate.sh`: Generate the butane config and therefore the ignition one. Copy at the right place also.
- `rebuild_node.sh`: Force boot on PXE and the wanted node (doing some mandatory prerequisite actions).
- `rebuild_cluster.sh`: Force boot on PXE and restart all the nodes.

# K3s install

## Cilium

## API HA

I needed the following for my cluster :

- A single point of contact for the API
  - When I want to reach it for API calls or a node joining the cluster
- Full availability if a node come down
  - That eliminates DNS round robin for instance

That why I chose Kube-VIP. It's a daemonset that is deployed during the creation of the cluster.  
Kube-VIP uses a leader election mechanism (using Kubernetes Lease or Raft). The elected leader owns and advertises the VIP.

If the leader node goes down:

- Leader lease expires
- Another node is elected
- The new node starts advertising the VIP
- Traffic fails over seamlessly
