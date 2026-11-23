

# Network Like an Operative
### Azure Network Traffic Analysis | Private Endpoint Connectivity Investigation

**Investigation Type:** Network Trace / Technical Ticket Response  
**Platform:** Microsoft Azure  
**Access Level:** Reader (Read-Only)  
**Focus:** Network Security Groups, User-Defined Routes, Private Endpoints, Private DNS, and Traffic Routing

---

## 1. The Ticket

> **Application Team:** "Our new service can reach the customer-data storage account, but I don't understand why or how. Can someone walk me through the network path?"

**Objective:** Trace the application's network path from its workload subnet to an Azure Storage account and explain how connectivity is possible despite a forced-egress route and disabled public access.

The investigation examines five configuration layers to determine how the application resolves, routes, and reaches the storage service.

---

## 2. The Network Trace

### Hop 1: The VNet — Identifying the Source

**What I checked:**

I reviewed the virtual network `vnet-lab-chap5` and inspected its configured subnets to figure out the source network and identify the security and routing controls that belong to it.

**What I found:**

The VNet contains two subnets:

| Subnet | Address Range | Purpose |
|---|---|---|
| snet-workload | 10.60.1.0/24 | Application workload |
| snet-appliance | 10.60.100.0/24 | Virtual network appliance |

The `snet-workload` subnet has two resources:

- Network Security Group: `nsg-lab-workload`
- Route Table: `rt-lab-workload`

**What this means:**

The workload subnet provides the starting point for the traffic investigation. Its associated NSG and route table establish the network controls that must be evaluated to understand the application's connectivity.

**Evidence:**

![Azure VNet Subnet Configuration](Images/NETWORKLIKEANOPERATIVE1.png)


<sub><i>Figure 1. Azure VNet subnet configuration showing the workload and appliance subnets, their IPv4 address ranges, and the security and route table associations.</i></sub>

---


### Hop 2: The NSG — Evaluating Traffic Restrictions

**What I checked:**

I examined `nsg-lab-workload` to identify the security controls for traffic entering and leaving the workload subnet.

**What I found:**

The NSG contains two custom inbound rules:

| Priority | Rule | Source | Port | Action |
|---|---|---|---|---|
| 310 | allow-vpn-https | 203.0.113.50 | TCP 443 | Allow |
| 4000 | deny-inbound-default | Internet | Any | Deny |

The first rule permits HTTPS traffic from one specified source address `203.0.113.50`.

The second rule denies inbound traffic originating from addresses classified under Azure's `Internet` service tag.

The NSG also retains Azure's default security rules, including `AllowVnetInBound` and `AllowVnetOutBound`.

There are no custom outbound security rules configured. Instead, the NSG retains three default outbound rules:

- `AllowVnetOutBound` (65000): Allows traffic between addresses matching the `VirtualNetwork` service tag.
- `AllowInternetOutBound` (65001): Allows outbound traffic to destinations matching the `Internet` service tag.
- `DenyAllOutBound` (65500): Denies outbound traffic that does not match a higher-priority rule.

**What this means:**

The custom inbound rules restrict access from the Internet, while the default outbound rules permit matching traffic within the virtual network and to Internet destinations.

The private endpoint address `10.60.1.62` falls within the workload subnet's `10.60.1.0/24` address range. Traffic to this address can match the NSG's default `AllowVnetOutBound` rule, which has a higher priority than `DenyAllOutBound`.  

The NSG determines whether traffic is allowed or denied, while Azure's effective routing configuration determines the packet's next hop.

**Evidence:**

![Azure NSG Security Rules](Images/NETWORKLIKEANOPERATIVE2.png)

<sub><i>Figure 2. NSG configuration showing custom inbound rules and Azure's default inbound rules. Three default outbound rules are also configured but are not visible in this screenshot.</i></sub>

---

### Hop 3: The Route — Investigating Forced Egress

**What I checked:**

I checked `rt-lab-workload` to determine how the outbound traffic from the subnet is routed and which device is designated as its next hop.

**What I found:**

The route table contains one user-defined route:

| Configuration | Value |
|---|---|
| Route Name | force-egress |
| Destination Prefix | 0.0.0.0/0 |
| Next Hop Type | Virtual appliance |
| Next Hop IP | 10.60.100.4 |
| Associated Subnet | snet-workload |

The default route forwards matching outbound destinations to a virtual appliance.

Azure normally creates a more-specific `/32` system route for private endpoints. In this case, the private endpoint's assigned address is `10.60.1.62`, corresponding to the host route `10.60.1.62/32`.

**What this means:**
A `0.0.0.0/0` route matches every IPv4 destination, but it is the least specific possible IPv4 prefix.

Azure uses longest-prefix matching to select a route. The private endpoint's `10.60.1.62/32` system route is more specific than the `0.0.0.0/0` user-defined route (UDR) and therefore takes precedence when both are applicable.

As a result, traffic destined for the private endpoint can bypass the virtual appliance at `10.60.100.4`, despite the workload subnet's forced-egress configuration.  

**Note:** Private endpoint route-table network policies can modify this behavior.

**Investigation limitation:** The route table confirms that the forced-egress UDR is configured. However, the effective route selected for traffic to `10.60.1.62` was not independently verified through effective-route diagnostics or traffic captures.  

**Evidence:**

![Azure Route Table Configuration](Images/NETWORKLIKEANOPERATIVE3.png)

<sub><i>Figure 3. User-defined route force-egress directing default IPv4 traffic to virtual appliance 10.60.100.4, with the workload subnet association visible.</i></sub>

---

### Hop 4: The Private Endpoint — Establishing Private Connectivity

**What I checked:**

I inspected the DNS configuration of the private endpoint `pe-lab-storage` to determine how the storage service is represented inside the virtual network.

**What I found:**

The private endpoint configuration identifies:

| Property | Value |
|---|---|
| Private Endpoint | pe-lab-storage |
| Private IP | 10.60.1.62 |
| Storage FQDN | stmadhatlabchap5.blob.core.windows.net |
| Private DNS Zone | privatelink.blob.core.windows.net |

The assigned address `10.60.1.62` belongs to the `10.60.1.0/24` workload subnet.

The endpoint provides a private network interface through which resources in the VNet can access the Azure Blob Storage service using Private Link.

**What this means:**

The storage account does not need to expose a public endpoint for the application to connect through an approved private endpoint.

The private endpoint serves as the private network entry point to the storage service, while Azure Private Link provides connectivity over Microsoft's network infrastructure.

The lab scenario states that public network access is disabled. This specific setting was not independently captured in the screenshots.

**Evidence:**

![Azure Private Endpoint DNS Configuration](Images/NETWORKLIKEANOPERATIVE4.png)

<sub><i>Figure 4. Private endpoint DNS configuration correlating the storage service FQDN with its assigned private address, 10.60.1.62.</i></sub>

---

### Hop 5: The Name — Resolving the Private Address

**What I checked:**

I inspected the record sets associated with the private DNS zone `privatelink.blob.core.windows.net` to determine how the storage hostname resolves to its private endpoint address.

**What I found:**

The DNS zone contains the following A record:

| Record Name | Record Type | IP Address | TTL |
|---|---|---|---|
| stmadhatlabchap5 | A | 10.60.1.62 | 10 seconds |

The record matches the private IP address assigned to `pe-lab-storage`.

**What this means:**

The application can request the storage account using its standard hostname rather than hardcoding an IP address.

Azure's storage DNS configuration uses the public storage hostname and its Private Link alias. When the private DNS zone is properly linked or accessible through DNS forwarding, the lookup returns the private endpoint address.

This allows the application to discover the private network destination dynamically.

The matching DNS record and endpoint configuration support the intended resolution path. A DNS query executed from the workload would independently confirm the actual result.

**Evidence:**

![Azure Private DNS Zone Records](Images/NETWORKLIKEANOPERATIVE5.png)

<sub><i>Figure 5. Private DNS zone A record mapping the storage account name to the private endpoint address 10.60.1.62.</i></sub>

---

## 3. Network Traffic Visualization

The following diagram illustrates the application's expected connection path and the relevant network controls.

```mermaid
flowchart TD
    A["Application / Workload Subnet<br/>Source of traffic"]
    B["DNS Resolution<br/>Public CNAME to Private Link<br/>Private A record returned"]
    C["NSG Evaluation<br/>Default VNet outbound allow"]
    D["Azure Route Selection<br/>Longest prefix match"]
    E["Private Endpoint<br/>Private Link entry point"]
    F["Azure Blob Storage<br/>Private service access"]
    G["Virtual Appliance<br/>Default egress path"]

    A -. "Resolve hostname" .-> B
    B -. "Return private IP" .-> A
    A --> C
    C --> D
    D -->|"Endpoint /32 selected"| E
    E --> F

    D -. "Other destinations using /0" .-> G

    H["Endpoint /32 is more specific<br/>than default UDR 0.0.0.0/0"]
    H -.-> D

    style A fill:#17365D,color:#fff,stroke:#457B9D
    style B fill:#274C77,color:#fff,stroke:#457B9D
    style C fill:#274C77,color:#fff,stroke:#457B9D
    style D fill:#8B5E34,color:#fff,stroke:#C58C46
    style E fill:#28745C,color:#fff,stroke:#40916C
    style F fill:#28745C,color:#fff,stroke:#40916C
    style G fill:#8A3037,color:#fff,stroke:#BC4749
    style H fill:#FFF3CD,color:#333,stroke:#D4A017
```

<sub><i>Figure 6. Logical network traffic flow showing DNS resolution, NSG evaluation, route selection, and private endpoint access. DNS is a preliminary lookup, not a data-plane hop. The virtual appliance represents the alternative default route.</i></sub>

---

## 4. Key Finding — Longest Prefix Matching

The most important discovery was understanding why a forced-egress route does not necessarily guarantee that every outbound packet passes through the network appliance.

Consider the following two routes:

| Route | Prefix | Routing Function |
|---|---|---|
| Default UDR | 0.0.0.0/0 | Sends matching traffic through the virtual appliance |
| Private Endpoint Host Route | 10.60.1.62/32 | Identifies one specific destination |

Azure prioritizes the longest matching address prefix.

A `/32` matches one IPv4 address, whereas a `/0` matches every IPv4 address.

Therefore, when both routes apply to traffic destined for `10.60.1.62`, the `/32` route takes precedence.

**Investigation takeaway:**

A forced-tunnel default route should not be interpreted as proof that all outbound traffic undergoes firewall inspection.

Route specificity and private endpoint network policies must also be considered.

---

## 5. The Answer — Reply to the Application Team

> **To: Application Team**
>
> **Subject: Storage Connectivity Investigation — Network Path Explained**
>
> I reviewed the application's Azure network configuration and identified how the service can reach the storage account using private connectivity.
>
> The application uses the storage account's standard hostname, which resolves through the configured Private Link DNS path to the private endpoint address `10.60.1.62`.
>
> The endpoint is located within the workload subnet's address range. The associated NSG contains no custom outbound restrictions, and its default VNet outbound rule permits matching private traffic.
>
> Although the workload subnet has a default route through virtual appliance `10.60.100.4`, Azure's more-specific private endpoint host route takes precedence over the `0.0.0.0/0` forced-egress route.
>
> Consequently, the application can use Azure Private Link to reach the storage service without relying on the storage account's public endpoint.
>
> This explains the connectivity reported in the ticket. The analysis is based on the existing Azure configuration; confirming the active route and inspection behavior would require additional validation.

---

## 6. What I Would Flag

**Potential security concern:**

The private endpoint routing configuration may allow storage-bound traffic to bypass the virtual appliance's normal inspection path.

I would recommend confirming with the network security team whether this routing behavior is intentional and whether alternative inspection or monitoring controls are in place.

Before closing a similar production ticket, I would also verify the effective route, DNS resolution from the workload, private endpoint approval status, and any additional security controls.

No configuration changes were performed during this investigation.

---

## 7. What I Learned

**Lesson 1 — Routing depends on prefix specificity.**

Understanding that Azure normally creates a /32 host route for private endpoints changed how I interpret forced-egress configurations. I learned that a 0.0.0.0/0 route does not necessarily mean all outbound traffic passes through the designated firewall, since more-specific routes can take precedence. This is important when planning internal network traffic and configuring routing tables to ensure traffic follows the intended paths and security controls.  


**Lesson 2 — Network connectivity requires multiple layers to work together.**

I learned how DNS, NSGs, routing tables, and private endpoints work together to establish network connectivity. DNS resolves hostnames to IP addresses, NSGs control permitted traffic, and routing determines the path packets take. I also learned that each component serves a different purpose and does not necessarily require its own IP address. Understanding IP ranges, route specificity, and Azure's default configurations is essential when designing network traffic paths. This investigation showed me how default and custom configurations interact and why they must be reviewed to prevent unintended traffic behavior or security bypasses.  

### What Surprised Me

Initially, I assumed that the 0.0.0.0/0 forced-egress route meant all outbound traffic would be directed through the virtual appliance at 10.60.100.4, including traffic to the storage account. What surprised me was learning that Azure normally creates a more-specific /32 host route for private endpoints, allowing traffic to bypass the default route. This challenged my understanding of forced-egress routing and taught me that a default route does not guarantee all outbound traffic passes through the firewall. It also reinforced the importance of examining effective routes when investigating network traffic and verifying that security controls work as intended.

---

## 8. Investigation Outcome

The investigation produced a configuration-based explanation of the application's private storage connectivity using read-only access.

**Skills demonstrated:**

- Azure Virtual Network and subnet analysis
- Network Security Group rule interpretation
- User-defined route evaluation
- Longest-prefix routing analysis
- Azure Private Endpoint and Private Link inspection
- Private DNS record correlation
- Security configuration analysis using Reader access
- Technical documentation and cross-team communication

**Investigation limitation:** This was a configuration trace, not a packet capture or a verified production traffic test.

---

## 9. References

- [Microsoft Learn — Azure Virtual Network Traffic Routing](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview)
- [Microsoft Learn — Network Security Groups](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)
- [Microsoft Learn — Azure Private Endpoint DNS Configuration](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns)
- [Microsoft Learn — Azure Storage Private Endpoints](https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints)
- [Microsoft Learn — Private Endpoint Network Policies](https://learn.microsoft.com/en-us/azure/private-link/disable-private-endpoint-network-policy)
- Mad Hat Labs — Network Like an Operative, Chapter 5 Lab
