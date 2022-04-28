#Using ExpressRoute Direct for customers with high security needs and in highly regulated industries

[Introduction](#introduction)
[Isolate traffic](#isolate-traffic)
[Encryption](#encryption)
[Guarantee bandwidth for applications and environments](#guarantee-bandwidth-for-applications-and-environments)
[Solution-architecture for customers in a highly regulated industry](#solution-architecture-for-customers-in-a-highly-regulated-industry)
<!---
[High availability and geo redundancy]
[Internet vs. MAPS vs. ExpressRoute]
[Availability Zones]
[AVS - Azure VMware Solution]
[Cost optimization]
[Overprovisioning]
[Limitations]
[Limits]
--->

##Introduction
Customers with high security needs or in regulated industries ask how they can meet their requirements for traffic isolation, encryption and dedicated bandwidth when connecting to Azure.
Based on my experience in many ExpressRoute Direct engagements with customers, I wanted to share my findings how ExpressRoute Direct can help to fulfil their requirements.
As a conclusion, I'll bring pieces together to show how customers in a highly regulated industry uses ExpressRoute Direct to fulfil their needs.

---

##Isolate traffic 
A common request is to (physically) isolate traffic between different environments. Eg. Dev environment should be (physically) isolated from prod environment.
Although Microsoft encourages customers to not adopt these on-premises practices to the cloud and instead use a more agile cloud approach, like usage of Hub and Spoke with Azure Firewall in the Hub, or use native features in vWAN for traffic isolation, customers might have regulatory requirements or might be mandated by their InfoSec policies to strictly isolate environments in the cloud and between on-premises and the cloud.
There are multiple ways to achieve this, let me explain the most common ones:
#####Build IPSec tunnels towards each Environment
IPSec VPN tunnels are built between OnPremises VPN devices and Azure virtual network VPN gateway. Whether Internet connectivity, Microsoft Azure Peering Service or Microsoft Peering on top of an ExpressRoute is used between OnPremises and Azure virtual network VPN gateway, depends on the requirements that exist in regards to latency, jitter and also available bandwidth on the path.

<img src="resources/Isolation-VPN-Tunnel.png" width=800></br>
#####*Pros:*
- Encryption between OnPremises Firewall / VPN Device and Azure Virtual Network Gateway
- Azure native approach
#####*Cons:*
- Cost
- Complexity
- Missing insights
- Bandwidth limitations (based on the VPN device / VPN gateway you're using)
#####Build Overlay network to each environment
This is a similar approach to the one before, but instead of native Azure VPN gateways, 3rd party NVAs will be used.
The difference is the feature set that 3rd party NVAs can offer. These features can reach from SDWAN (eg. multiple links, path selection), up to deep insights into communication patterns and traffic. Also multicloud capabilities, operations and troubleshooting are important additions. Vendors are for example Aviatrix, Fortinet and Palo Alto.

<img src="resources/Isolation-Overlay.png" width=800></br>
#####*Pros:*
- Flexibility
- Scalability (depending on Architecture)
- Cost (depending on Architecture)
- Encrpytion (depending on Architecture)
#####*Cons:*
- Cost (depending on Architecture)
#####Use ExpressRoute per Environment
This architecture builds on isolation with dedicated ExpressRoutes per environment. While customers of course can use multipe ExpressRoutes from a single or multiple providers, using a cloud exchange provider (like DE-CIX, Equinix, InterXion or Megaport) is the most flexible and scalable model.
For every environment, a dedicated ExpressRoute Circuit is used, which in turn means, isolation will happen on layer-2 (still L3 will be used for routing and this is not a L2 extension/stretch!).
Besides the physical isolation, also a dedicated bandwidth is given for each ExpressRoute. This will be discussed later in this document.

<img src="resources/Isolation-Cloudexchange.png" width=1200></br>
#####*Pros:*
- Small bandwidth chunks available (50Mbit/s)
- Dedicated bandwith
- Limits and Quotas (depending on Architecture)[^1]
- Azure native approach
#####*Cons:*
- no native encryption available (need to add VPN and/or Overlay on top of ER)
- Cost
- Complexity
#####Use ExpressRoute on top of ExpressRoute Direct
This is again a variant, in this case of the previous architecture, and it combines the flexibility of a cloud exchange with a dedicated connection to Microsoft by using ExpressRoute Direct.
ExpressRoute Direct is like a container for ExpressRoute circuits. When you create an ExpressRoute Direct, you connect directly to Microsoft wihtin an ExpressRoute peering location (Microsoft will issue a so called LoA (Letter of Authorization) to establish a direct CrossConnect in the peering location to Microsofts Enterprise Edge Routers). There's no connectivity provider inbetween, you become the ExpressRoute provider.
After you've established the physical connectivity to Microsofts Enterprise Edge Routers, changed the "admin state" to enabled and the your fiber is lit, you can start to create ExpressRoute Circuits on top of the ExpressRoute Direct.

<img src="resources/Isolation-ERDirect.png" width=1200></br>

#####*Pros:*
- Encryption - there's a separate topic later in this document
- Bandwidth - up to 100 Gbit/s
- Flexibility
- Azure native approach
- Direct connection to Microsoft
#####*Cons:*
- Cost
- Complexity

####Summary
As shown, there are different ways to isolate traffic. Depending on customers needs, 

---

##Encryption
A common request is to encrypt the traffic between customers network and Microsoft Azure.
While general advice is to use only encrypted protocols, customers often face challenges ensuring that they're only using encrypted protocols or simply using protocols that doesn't offer built-in encryption capabilities.
Using a VPN or encryption in the underlay is a common practice. Still there are a couple of ways to do this.
Below are the most common ways to encrypt traffic via VPN or the underlying infrastructure (please note that for simplicity I skipped VPN redundancy, in real life, it is a good practice to crossconnect VPN tunnels to maximize throughput and minimize effects when a VPN device/gateway fails/is upgraded).

:exclamation: Before moving on, let me share a word on encryption on the Microsoft backbone. Microsoft encrypts all data in transit that leaves their datacenters and crosses public grounds. This is publicly disclosed and written here: https://docs.microsoft.com/en-us/azure/security/fundamentals/encryption-overview#encryption-of-data-in-transit . 


####IPSec VPN over Internet
The most common approach used to encrypt traffic is using an IPSec tunnel. While this is a low hanging fruit, since you can create the tunnel over the internet, it comes with limitations like limited bandwidth (although Microsoft VPN gateways support up to 10 GBit/s, it's only reached by using multiple tunnels and using BGP with ECMP to load balance the traffic across the tunnels. Also, due to the fact that the IPSec tunnel limit is 1 / 1,25 Gbit/s, your single flow performance is limited by this) and IPSec overhead.
If you use a 3rd party NVA, you might overcome this limit and get a higher throughput.

<img src="resources/encryption-Internet-VPN.png" width=1200></br>

#####*Pros:*
- Fast adoption when using Internet and Azure VPN gateways
- Failover options with active/active GWs and BGP
- Cost
#####*Cons:*
- Limited bandwidth / tunnel performance
- Uses public Internet for transport -> might have an impact on bandwidth and / or latency. No SLA
- IPSec compatibility with existing devices
- Complexity (multiple tunnels, BGP, ECMP)
- Troubleshooting of IPSec
- Key rotation complexity (if key rotation is required, which is likely in highly regulated industries)

####IPSec VPN over Microsoft peering on ExpressRoute
In this architecture private connectivity to Microsoft using ExpressRoute and Microsoft peering is used with an IPSec tunnel on top of it. So you'll get the advantages of private connectivity to Microsoft using ExpressRoute, like dedicated bandwidth (of transport, IPSec tunnel limits stay the same).

<img src="resources/encryption-VPN-ER-mspeering.png" width=1200></br>

#####*Pros:*
- Failover options with active/active GWs and BGP
- Dedicated private connectivity
- Bandwidth
#####*Cons:*
- Limited bandwidth / tunnel performance
- IPSec compatibility with existing devices
- ExpressRoute Microsoft Peering must be configured
- Complexity (multiple tunnels, BGP, ECMP)
- Troubleshooting of IPSec
- Key rotation complexity (if key rotation is required, which is likely in highly regulated industries)

####IPSec VPN over private peering on ExpressRoute
While Microsoft peering on ExpressRoute gives you great flexibility regarding access to Azure PaaS, it comes with a higher complexity (eg. NAT) and also with some requirements (eg. need for public IPv4 IPs that are owned by the company, or that the company is allowed to use, and that are not propagated to the Internet) that makes it sometimes hard to implement.
Microsoft implemented the option to use an Azure VPN Gateway behind a Microsoft ExpressRoute Gateway over the private peering (which is quite easy to configure).

<img src="resources/encryption-VPN-ER-privatepeering.png" width=1200></br>

#####*Pros:*
- Failover options with active/active GWs and BGP
- Dedicated private connectivity
- Bandwidth
#####*Cons:*
- Limited bandwidth / tunnel performance
- IPSec compatibility with existing devices
- Complexity (multiple tunnels, BGP, ECMP)
- Troubleshooting of IPSec
- Key rotation complexity (if key rotation is required, which is likely in highly regulated industries)

####Variants of above encryption options
Both architectures might benefit from a substitution of Microsoft VPN gateways with 3rd party NVAs. These solutions might come with a higher cost, so you need to take a closer look at the cost associated with 3rd parties. Nevertheless, depending on your requirements regarding troubleshooting, throughput and ease of use (complexity for operation of NVAs in Azure might be higher, so compare solutions carefully), you might find a good option here.

####End-to-End IPSec encryption between hosts (in combination with ExpressRoute and private peering)
Customers looking for truly end-to-end encryption, should think about building IPSec tunnels between hosts. While this is the most complex architecture, and for sure the one with the highest operational overhead, especially when you have different operating systems, it's the most secure approach since you don't need to worry about any encryption on the "road".

<img src="resources/encryption-end-to-end.png" width=1200></br>

#####*Pros:*
- True end-to-end encryption
- Scalability due to distribution of encryption
#####*Cons:*
- Complexity
- Interop between different operating systems IPSec implementation
- Management at scale if using different operating systems


####L2 Encryption on ExpressRoute Direct with MACSec (802.1AE)
When it comes to high speed encryption, all of the above mentioned methods suffer from limited bandwidth and additional overhead.
Especially when it comes to IPSec encryption beyond 10G, solutions get quite expensive and available virtual appliances in the cloud are rare and still limited.
Looking at the available options, Microsoft decided to offer MACSec (802.1az) encryption on ExpressRoute Direct. MACSec works on L2 and offers unmatched, near line rate, performance, also with 100G bandwidth. In addition, it has a good interoperability (Microsoft just added some configuration options to be compatible with solutions that doesn't offer flexibility in configuration).
MACSec encryption is offered only on ExpressRouter Direct, since this is the only solution that offers a direct connectivity via L1 between customer and Microsoft, which is needed for MACSec (there are partners that offer a transparent L2 transport that can be used to "extend" the physical part).
In this case, encryption is "divided" into two pieces. Customer encrypts traffic between his devices and Microsofts devices and Microsoft encrypts traffic on his backbone (see statement on encryption of Microsoft backbone above). Although Microsoft states this officially, customers must trust Microsoft (which is usually not a problem, because they're putting their workloads and data into Azure / Microsofts cloud, and trust should already be "documented" through available information, eg. from the TrustCenter).



<img src="resources/encryption-erdirect-macsec.png " width=1200></br>

#####*Pros:*
- Near line-rate performance, even with 100Gbit/s
- Minimal protocol overhead
- Scale
- Cost (compared to a "traditional" IPSec based encryption and especially when using switches for MACSec).
- Good options for key rotation
- Complexity is lower than with IPSec and it's 
- Interoperability of different vendors
- Flexibility of ExpressRoute Direct (see section above on "ExpressRoute on top of ExpressRoute Direct")
#####*Cons:*
- Cost for ExpressRoute Direct

----

###Guarantee bandwidth for applications and environments
Often customers want or need to dedicate a certain bandwidth to applications or environments. There are multiple reasons for this, for example, they want to ensure that the applications or environments doesn't interfere with each other and a single environment or application can't bring down other environments. Another example is the usage of AVS, where you want to have a dedicated bandwidth to ensure smooth migration of workloads without impacting the rest of your workloads running in Azure.
Since Azure, like other cloud providers, doesn't support natively QoS, rate limiting or traffic shaping, there's no way to achieve the requirement to limit the bandwidth.
Of course, you can build an overlay with 3rd party appliances that support Qos and / or rate limiting and traffic shaping, but let's look a little bit closer into ways that use Azure native capabilities to guarantee bandwidth for applications.
:exclamation: Application or environment is used as a synonym for a service that is running in a virtual network (VNet) in Azure. So eg. when talking about a PROD Environment, this is the synonym for applications that are running in the VNet (or VNets if Hub & Spoke is used).

####IPSec VPN
Let's start with the traffic path when using an IPSec VPN.
If you look at the traffic path between On-Premises and Azure, there are two components that are capable of rate limiting and traffic shaping. First the VPN Device On-Premises can be configured for traffic shaping, if it offers the option. Depending on the device you use and the feature set it offers, you have a granular control on the traffic.
The second option to provide some kind of rate limiting, is the VPN gateway in Azure. The VPN gateway in Azure comes in different "sizes", that support different bandwidth.

VPN Gateway sizes and bandwidth (excerpt, see full table [here](#https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-about-vpngateways#benchmark))
|Gateway Type|Generation|Bandwidth|
|:--:|:--:|--:|
|VpnGw1|Gen 1|650 Mbit/s|
|VpnGw2|Gen 1|1 Gbit/s|
|VpnGw2|Gen 2|1.25 Gbit/s|
|VpnGw3|Gen 2|2.5 Gbit/s|
|VpnGw4|Gen 2|5 Gbit/s|
|VpnGw5|Gen 2|10 Gbit/s|

In the example below, you can see that the PROD environment is using a VPN GW of type "VpnGW4" which offers 5 Gbit/s encryption performance, while the DEV environment uses a "VpnGW2", which offers only 1.25 Gbit/s encryption performance.
So the Bandwidth for each environment is limited by using different VPN GW types.

<img src="resources/bandwidth-vpn.png" width=1200></br>

####ExpressRoute
The second option to natively limit bandwidth between OnPremises and Azure, is to use ExpressRoute.
Since ExpressRoute offers different bandwidth, you can use that to rate limit the traffic and guarantee applications or environments a certain bandwidth.
Bandwidth of ExpressRoute circuit is rate limited to the configured ExpressRoute Circuit size on provider router or on customer CPE (in case of ExpressRoute Direct).

|Type|Circuit sizes|
|--|--|
|ExpressRoute| 50Mbit/s, 100Mbit/s, 200Mbit/s, 500Mbit/s, 1Gbit/s, 2Gbit/s, 5Gbit/s, 10Gbit/s|
|ExpressRoute on top of ExpressRoute Direct 10Gbit/s| 1Gbit/s, 2Gbit/s, 5Gbit/s, 10Gbit/s|
|ExpressRoute on top of ExpressRoute Direct 100Gbit/s | 5Gbit/s, 10Gbit/s, 40Gbit/s, 100Gbit/s|

</br>

The second option to limit the bandwidth is again the virtual network gateway, but not type VPN, instead the ExpressRoute Gateway type. There's a gotcha here, ExpressRoute gateways only limit the incoming traffic (from ExpressRoute) towards the VNet. There's no limit on the egress traffic.
|Gateway Type|Bandwidth|
|:--:|--:|
|Standard/ERGw1Az|1 Gbit/s|
|High Performance/ERGw2Az|2 Gbit/s|
|Ultra Performance/ErGw3Az|10 Gbit/s|
</br>

:exclamation: When you enable FastPath on the UltraPerformance/ErGw3Az, gateways are bypassed and thus bandwidth is not limited :exclamation:
</br>
<img src="resources/bandwidth-expressroute.png" width=1200></br>

####Summary on bandwidth limitation or guaranteeing bandwidth
As you can see there are some solutions to control bandwidth between OnPremises and Azure.
Nevertheless, not all solutions offer all options (or depend on your existing infrastructure), while ExpressRoute gateways only offer a limitation of ingress traffic towards VNet.
Leaving this, the best option is to use ExpressRoute circuits to limit the bandwidth and guarantee that the existing bandwidth is shared according to your needs and configuration.

----
###Solution-architecture for customers in a highly regulated industry
Now that you've learned about different options that fulfil requirements regarding isolating traffic, encryption and guaranteeing bandwidth for certain environments or applications, let's bring this all together to build an architecture that fulfils the mentioned reuqirements for a customer in a highly requlated industry.

The below architecture builds on ExpressRoute Direct and offers:
1. Traffic isolation
By creating ExpressRoute Circuits on top of ExpressRoute Direct, customer can create isolated, dedicated logical links to their environments. Each ExpressRoute ciruit on an ExpressRoute Direct is using a dedicated (customer defined) VLAN. Also, you can create a private peering and a Microsoft peering for each ExpressRoute Circuit, which gives additional flexibility. Mixing / adding ExpressRoute types, like "ExpressRoute local", can help optimize costs.
2. Encryption
MACSec is the foundation for high-speed encryption on layer 2. It's also easy to implement and very robust. Due to the fact, that Microsoft handles MACSec keys independently for primary and secondary link, key rotation is possible without downtime (just loosing redundancy while keys are changed).
3. Guaranteeing bandwidth
Since there are no ways to limit bandwidth or do traffic shaping in Azure, best option is to use ExpressRoute Circuits of different sizes and use the size you need. In the example below, all PROD environments have a bandwidth of 5GBit/s and all non-prod or DEV environments have a bandwidth of 2Gbit/s. Since you can overprovision bandwidth on an ExpressRoute Direct with a ratio of 2:1, there's still 6Gbit/s left on the ExpressRoute Direct in the example below.

<img src="resources/solution-summary.png" width=1200></br>

<!---
VLAN Tagging 
Dot1q vs Q-in-Q

####Best practices
- build dashboards for monitoring
- impement 
- use BFD
- MACSec key rotation
- Deploy ExpressRoute Directs in multiple subscriptions
- Plan bandwidth usage and overprovisioning of bandwidth
  It sounds obvious, but I've seen many customers failing on this piece.
  As you may know, you can overprovision an ExpressRoute Direct with a ration of 2:1. That means, you can create ExpressRoute Circuits up to twice the bandwidth of the ExpressRoute Direct. For example, if you have a 100G ExpressRoute Direct, you can create ExpressRoute Circuits with a total bandwidth of 200G on top of it. Eg.
  1 x 100Gbit/s
  10 x 10Gbit/s
  :exclamation: Please keep in mind that you need to raise the limit of ExpressRoute circuits per subscription via a support request if you want to create more than 10 ExpressRoute circuits (which is the default number, see also later in the section on limits).
  To make the maximum out of your ExpressRoute Direct, it's a good approach to capture the bandwith usage in a spreadsheet like in the simple example below. This not only helps you in planning, but also in troubleshooting situations where these informations might be handy. 
  | Gbit/s | ExpressRoute Direct | ExpressRoute Circuit | Type | Description
  |--|--|--|--|--|
  |1|ExpressRoute Direct Equinix FR7| FRA-Prod-34 | Standard
  |2|ExpressRoute Direct Equinix FR7| FRA-Prod-34 | Standard
  |3|ExpressRoute Direct Equinix FR7| FRA-Prod-34 | Standard
  |4|ExpressRoute Direct Equinix FR7| FRA-Prod-34 | Standard
  |5|ExpressRoute Direct Equinix FR7| FRA-Prod-34 | Standard
  |6|ExpressRoute Direct Equinix FR7| FRA-Prod-34 | Standard
  |7|ExpressRoute Direct Equinix FR7| FRA-Prod-34 | Standard
  |8|ExpressRoute Direct Equinix FR7| FRA-Prod-34 | Standard
  |9|ExpressRoute Direct Equinix FR7| FRA-Prod-34 | Standard
  |10|ExpressRoute Direct Equinix FR7| FRA-Prod-34 | Standard
  |11|ExpressRoute Direct Equinix FR7| FRA-Dev-34 | Local
  |12|ExpressRoute Direct Equinix FR7| FRA-Dev-34 | Local
  |13|ExpressRoute Direct Equinix FR7| FRA-QA-34 | Standard
  |14|ExpressRoute Direct Equinix FR7| FRA-QA-34 | Standard
  |15|ExpressRoute Direct Equinix FR7| FRA-LZ-ASIA | Premium
  |16|ExpressRoute Direct Equinix FR7| FRA-LZ-ASIA | Premium
  |17|ExpressRoute Direct Equinix FR7| FRA-LZ-ASIA | Premium
  |18|ExpressRoute Direct Equinix FR7| FRA-LZ-ASIA | Premium
  |19|ExpressRoute Direct Equinix FR7| FRA-LZ-ASIA | Premium
  |20|Spare|Spare|||



- 

####Limitations
- ExpressRoute Circuits beyond 10Gbit/s created on top of an ER Direct 100G
- 

####Limits
- number of VNets attached to ER Circuit
- Maximum number of ExpressRoutes
- Raise number of maximum circuits per subscription
- 100G maximum of 20 ExpressRoutes (40 with oversubscription)
- 10G maxiumum of 10 ExpressRoutes (20 with oversubscription)


References and Footnotes
[^1] Azure subscription and service limits, quotas, and constraints : https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits
--->