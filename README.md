
# Azure Storage, ADLS Gen2 and SQL DB Over ExpressRoute Private-Peering 

Currently, Microsoft Azure Storage, ADLS Gen2 and SQL DB services offer only public IP endpoints for device and
client connectivity.  While all communication with Azure Storage and SQL require an encrypted TLS/SSL
channel, there are customers who prefer device communication with the Azure Storage and SQL service to occur
over a private connection. 

There are several important use cases where Azure Storage, ADLS Gen 2and SQL DB would benefit from offering a private
endpoint to devices and clients:

-	Private traffic though ExpressRoute (e.g., factory devices with secure private IPs that use MPLS for Cloud connectivity)
-	Private traffic through a VPN (e.g., remote sensors that use P2S for high security)
-	Devices requiring internal DNS resolution of a PaaS endpoint

# The Solution – NGINX as a Private Azure Storage, ADLS Gen2 and SQL DB Gateway

NGINX (pronounced “engine-x”) is a free, open-source, high-performance HTTP server and reverse
proxy.  It is well known for its high performance, stability, rich feature set, simple configuration, and
low resource consumption.

Its applicability to solving this private gateway challenge is as follows:

-	NGNIX is used as an HA scale-out tier that provides a private IP endpoint for Azure Storage and SQL DB clients and devices.
-	NGINX is deployed as VMs, VMSS, containers, or even within Azure Service Fabric to fit a variety of scale and automation needs.
-	NGINX can be deployed as a scalable HA tier in the VNet that will offer private IP access to both IaaS and on-prem clients, then         will securely forward traffic to the Azure Storage, ADLS or SQL DB using ServiceEndpoints. More on this below.

TCP Load Balancing in NGINX

While NGINX is primarily used as a web proxy and HTTP application load balancer, NGNIX also
allows for TCP layer load balancing. TCP load balancing mode is configured in a stream block and
does not need to be configured with TLS certs or keys. In TCP mode, NGINX will forward the TLS
handshake through to the target server, or upstream server, in the NGINX terminology.  For this
architecture, we will use TCP load balancing in NGINX to keep the configuration simple and to
allow secure forwarding for other TCP services.

Other advantages include:

-	TCP load balancing helps increase performance and lower cost because NGINX does not need to perform DPI or deep packet      
        inspection, at the application layer.
-	NGINX can still use an FQDN as an upstream server in the stream block and will resolve DNS against the FQDN before forwarding 
        the connection. In the free, basic version of NGINX, DNS resolution will occur each time NGINX is started. 
-	NGINX Plus can dynamically resolve the upstream FQDN with a variable TTL setting in TCP mode. It prevents dynamic changes of the 
        underlying PaaS Public IP from requiring a restart of the NGINX service.  For a more robust design, you can consider upgrading 
        to receive this functionality.

# The Essential Architecture

Virtual Network (VNet) Service Endpoints extends your virtual network private address space, and
the identity of your VNet, to multi-tenant Azure PaaS services over a special NAT tunnel in the
Azure fabric.  Service Endpoints allow you to secure your critical Azure service resources to only
your virtual networks. Traffic from your VNet to the Azure PaaS service always remains on the
Microsoft Azure backbone network.

The key here is the ability to deploy Azure Storage and SQL DB with Service Endpoints, while also making these
private PaaS endpoints available over the ExpressRoute private peering. The current limitation of
Service Endpoints is that it is not accessible to on-premise resources.  The NGINX Gateway solution
detailed here provides a proxy solution to allow connectivity from on-premise.  A native solution to
extend a private endpoint on-premise is on the roadmap called Private-link. 

The following diagram illustrates on-premise to cloud communication architecture secured via the
NGINX private gateway hosted in Microsoft Azure. 

![alt text](https://github.com/jgmitter/images/blob/master/VisioImage2.png)



Network

Resource group with a VNet and a minimum of two subnets:
-	GatewaySubnet (contains Azure ExpressRoute Gateway, /27 min, /26 recommended)
-	SUBNET_NGINX (contains N NGINX VMs and Azure Internal Load Balancer)

NSGs to provide security:
-	Only allow private IP ranges to SUBNET_NGINX and service-tag of Azure Storage, ADLS & Azure SQL on service destination ports 443 (HTTPS), and 1433 (SQL).
-	NSGs are used to enhance network security in the load balancing and NGINX subnet tiers, exposing just those services needed, and restricting inbound traffic to private ranges only.
-	Other VNet ranges can be added to inbound/outbound rules for increased security within the VNet (not shown below)


Example of inbound NSG rules for SUBNET_NGINX

![alt text](https://github.com/jgmitter/images/blob/master/inbound.png)

Example of outbound NSG rules for SUBNET_NGINX

![alt text](https://github.com/jgmitter/images/blob/master/outbound.png)

Traffic flow

-	On-Premise devices will resolve an FQDN for their Storage, ADLS, or SQL DB service, with the A record as the private IP of the 
        Azure Internal Load Balancer (ILB). The real FDQN need only be known to NGINX. This allows for internal/custom DNS applications.
-	Contoso will connect to the ILB via ExpressRoute Private-Peering
-	NSGs applied on subnet will drop any packets that do not match either the private IP of the client device or the destination  
        service required (HTTPS/SQL-DB). 
-	The ILB will load balancing incoming HTTPS or SQL DB traffic across the NGINX tier.
-	NGINX tier will proxy incoming TCP connections to Azure Storage, ADLS and/or SQL DB as an upstream server.
-	Azure Storage, ADLS, and/or SQL DB will check its IP filer and allow the incoming packet if it is a match against the whitelist.         Else it will drop the packet.
-	The response will flow back on the open TCP connection, which will be statefully mapped back to the NIC of the NGINX server that         owns that connection flow.

Azure Storage, ADLS, and SQL DB

-	Azure Storage, ADLS and SQL DB are configured as normal.
-	NGINX will use FQDN of Storage, ADLS and/or SQL DB as target upstream server in config
-	Storage, ADLS and SQL DB VNet Rules (IP Filters) will be enabled to only allow traffic from SUBNET_NGINX and deny Internet outbound (0.0.0.0/0)
-	Storage, ADLS and SQL DB will negotiate TLS Handshakes and certificate auth directly with Storage, ADLS and SQL DB clients and devices. NGINX is using TCP pass-through and does not need to see encrypted payload of TCP segments
-	NGINX will support web data stream in TCP pass-though mode via HTTPS/443; configuration is agnostic to HTTPS data stream.
-	Local DNS conditional logic or split DNS can be used to avoid TLS certificate errors.

For example, your local DNS can have a conditional policy, or split DNS policy,  to hand out  “10.1.3.50” as the A record for *.servicebus.windows.net, so that the fqdn request will continue to match the commonName value in the certificate, resulting in an error-free TLS handshake:

![alt text](https://github.com/jgmitter/images/blob/master/certificate.png)

ILB Tier
-	Azure Internal Load Balancer is ideal to provide a single private endpoint which can be used for internal DNS resolution.
-	Private FQDN of ILB can be anything customer needs
-	ILB provides great scale out and 99.95% uptime for a solution
-	ILB has BackEnd pool of “N” NGINX Ubuntu Servers (or use VMSS, not scoped here)
-	ILB will have two front-end rules: 443 for HTTPs, and 1433 for SQL DB
-	ILB uses two TCP probes for each load balancing rule: HTTPS, and SQL DB
-	The ILB Front-End IP should be a static IP and should originate from SUBNET_NGINX


Load-balancing rules:

![alt text](https://github.com/jgmitter/images/blob/master/lbrules.png)

Load-balancing probes:

![alt text](https://github.com/jgmitter/images/blob/master/probes.png)


NGINX Tier

-	Can be deployed on RHEL, Debian, Ubuntu, or SuSE
-	Four recommended VM tiers for production, scale UP paradigm (use VMSS for scale-out):
-	Low – Backend pool of 2 D2S_V3 VMs, 2 cores and 8 GB of RAM each
-	Medium – Backend pool of 3 D4S_v3 VMs, 4 cores and 16 GB of RAM each
-	High – Backend pool of 3 F8S VMs, 8 cores and 16 GB of RAM each
-	Ultra – Backend pool of 4 F16s VMs, 16 cores and 32 GB of RAM each
-	NGINX configs are set up to scale worker threads across multiple cores, use epoll kernel extensions, set connection pools again “ulimit –n,” allow instant, new connection handling, and have TCP_NOPUSH enabled.
-	VMSS can also be used as a powerful way to harness auto-scale and workload agility, but this is not scoped here.
-	Service Endpoints Enabled on your NGINX subnet to service “Storage and SQL”
-	Since Service Endpoints is enabled on your NGINX subnet, you will need NSGs to allow the NGINX subnet to open connections to the NSG Service Tags of Azure Storage and SQL DB of your region – see NSG section above.
-	Finally, if you are using a forced tunnel route, this is supported, because the ServiceEndpoint routes in the NGINX Subnet will be more specific that your default gateway route (0.0.0.0). Traffic between the NGINX subnet and Azure Storage, ADLS and/or SQL DB will flow directly over the private NAT tunnel in the Azure fabric. Enable Service Endpoint within NGINX Subnet (Shown as “nginxvmsssubnet” in the below example):


Enable Service Endpoint within NGINX Subnet (Shown as “nginxvmsssubnet” in the below example):

![alt text](https://github.com/jgmitter/images/blob/master/subnet.png)

Testing of Storage ADLS Gen2 and SQL DB:

Download the following tools to test the solution from a client perspective:
-	https://azure.microsoft.com/en-us/features/storage-explorer/
-	https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-2017


Sample NGINX Configuration for Azure Storage, ADLS Gen2 and SQL DB: https://github.com/jgmitter/storage-sql/blob/master/NGINX-Storage-SQL.conf.txt

A VMSS for deploying NGINX as a TCP Stream Broker



<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fjgmitter%2Fnginx-vmss%2Fmaster%2Fazuredeploy.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>

This template allows you to deploy a VM Scale Set of NGINX Ubuntu VMs using the latest patched version of Ubuntu Linux 18.04 LTS. These VMs are behind an internal load-balancer. Because the load-balancer is internal, you must first ssh into the jumpbox, then ssh from there into a specific VM behind the load balancer. To connect from the load-balancer to a VM in the scale set, you would go to the Azure Portal, find the jumpbox public ip, examine the NAT rules, then connect using the NAT rule you want. For example, if there is a NAT rule on port 50000, you could use the following command from the jumpbox:

ssh to your jumpbox, than:

ssh -p 50000 {username}@{loadbalancer-ip-address} to each NGINX VM to update its NGINX.conf



# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
