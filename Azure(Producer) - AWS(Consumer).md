

# Azure (Producer) to AWS (Consumer):
Here Azure Databricks acts as a data producer and AWS Databricks acts as the consumer. 
# Network Architecture:
![azure-to-aws.png](images%2Fazure-to-aws.png)
# Private Storage Network:
To ensure the storage account in Azure Databricks is private, disable public access on the storage account and create dedicated private endpoints between the workspace VNet and the storage account
# Cross-Cloud Connection Instruction:
 - Create a Virtual Network Gateway in Azure with the following properties
    - Gateway type: VPN
    - SKU: VPNGw2AZ
    - Virtual Network: The Vnet where Databricks workspace is deployed or a VNet which is peered to the databricks Vnet (Hub-Spoke model)
    - Gateway Subnet address Range: This will create a gateway subnet with the address specified, or alternatively select a pre-created gateway subnet
    - Generation: Generation2
    - Public IP: Create a new or select an existing IP. This IP is used on the AWS side when creating the customer gateway
    - Enable active-active mode: Disabled
    - Configure BGP: Disabled
    - Review and create
 - Switch to aws side and create a new Customer gateway with the following details:
    - Name: identifier name
    - Ip Address: Enter the public IP of the Azure Virtual Network Gateway form 
 - Create Virtual Private Gateway and give a name
 - Attach the Virtual Private Gateway to the AWS VPC where databricks workspace are deployed. It could alternatively be deployed on a Transit VPC which has Transit gateway which is then associated with one or more space VPC (which contains Databricks workspace) 
 - Create a Site-To-Site VPN Connection with the following details:
    - Name: name of the connection
    - Target gateway type: Select the Virtual Private Gateway created earlier
    - Customer Gateway: Select the Customer Gateway created earlier
    - Routing Options: Static
    - Static IP Prefix: The subnet on azure side which needs to be connected. In this case add the subnet CIDR of the private Link subnet which contains the private endpoint for the uc metastore
    - Create connection
 - Once the connection state changes to available, select the VPC connection and download the configuration. While downloading select Vendor and Platform as generic, Software as VendorAgnostic and IKE version as ikev2. The downloaded file contains the following details
    - The VPN Connection creates two tunnel
    - For each tunnel , note down the Outside IP Addresses→Virtual Private Gateway→IP Address
    - For each tunnel , note down the Pre-shared Key
    - These two details are required on the azure side when creating the Local Network Gateway in azure
 - Switch to azure side and create Local Network gateway with the following details:
    - Resource Group, Region, Name
    - Endpoint: IPAddress
    - IPAddress: enter one of the IP address from the downloaded configuration file in the previous step
    - Address Space: The CIDR of the aws VPC which has the Virtual Private Gateway (could be same as the Databricks VPC or different in a Hub Spoke model)
    - Configure BGP setting: No
    - Create 
 - Go to the Virtual Network Gateway and under Settings→Connections click Add and enter the following details
    - Resource Group
    - Connection type: Site-to-Site(IPSec)
    - Name & Region
    - Virtual Network Gateway: Select the one created previously
    - Local Network gateway : select the Local Network gateway created in the previous step
    - Shared Key: Enter the Pre-shared key for tunnel 1 from the downloaded configuration file
    - IKE Protocol: IKEV2
    - Create
 - Confirm status of connection showing as Connected from Azure side
 - Confirm status of tunnel 1 of the VPN connection on AWS side as Up
 - Repeat step 7-10 for tunnel two by creating another Local Network Gateway and Connection and entering details of tunnel 2 IP address and Pre-shared key. Verify connection status in azure and tunnel status in aws showing connected/up
 - The tunnel connection is now set up
 - The next step is to route traffic from aws cluster to the azure pl subnet
The databricks subnet on aws generally should be associated with a route table which redirects traffic destined for the internet towards Nat gateway ( for SCC cluster). The Nat gateway is associated to a route table which routes all internet traffic (0:0:0:0/) to internet gateway
 - In the Route table associated with the Nat gateway, click edit route and enter the below route details
    - Destination: the CIDR of the PL subnet on azure side
    - Target: Virtual Private Gateway and select the Virtual Private Gateway created earlier
    - This will ensure any connection to the IP address of the storage PL is routed to the Virtual Private Gateway
 - Now that the routing is also established, there is still one additional step required. Since the storage account is public disabled and only private endpoint is created, the private endpoint created in azure by default associates a private IP and also creates private dns zones and a A record in the zone to resolve storage account FQDN to its private IP. Now this works for a cluster created under Azure databricks as it can resolve the IP from the Azure DNS zone, the same is not available on AWS side
 - In order to resolve, this, go to Route 53 and create a hosted zone with the following details
    - Domain name: fqdn of the azure storage account (Ex: hsstoragene.dfs.core.windows.net), you could use a different zone name to keep it generic (Ex: dfs.core.windows.net)
    - Type: Private Hosted Zone
    - VPC Association: Select the aws vpc containing the databricks workspace configured
    - Create hosted zone
 - Select the zone and create A record with the following details:
    - Record name: empty as the zone contains the full fqdn of the storage account
    - Value: enter the private IP address of the private link NIC associated with the storage account in azure
    - Create record
 - Test connection:
    - Spin up a cluster in AWS workspace
    - Query the Azure shared table
    - The cluster should connect with the hosted zone associated with the VPC to resolve the name of the storage account to its private IP
    - The cluster should be able to access the storage account through the routes configured from subnet route→natgateway route→virtual private gateway→tunnel→local network gateway→storage account
