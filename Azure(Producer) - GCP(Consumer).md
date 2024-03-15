# Azure (Producer) to GCP (Consumer):
Here Azure Databricks acts as a data producer and GCP Databricks acts as the consumer. 
# Network Architecture:
![Azure-to-GCP.png](images%2FAzure-to-GCP.png)

# Private Storage Network:
Configure the storage firewall to block public access
create a private endpoint connection to Vnet used by the Azure Databricks workspace for a `dfs` sub-resource (link)

# Cross-Cloud Connection Instruction:
Create an HA VPN connection between Azure Vnet and Google Cloud VPC. (Refer to Google Cloud documentation for more details if you have doubts: link)
 - In the Azure portal, create a Virtual Network Gateway. Pay attention to the following configuration details:
   - Resource Group - same as the Vnet used by Databricks workspace
   - Region - same as the Vnet used by Databricks workspace
   - Gateway Type - VPN
   - VPN Type - Route-based
   - Virtual network - Vnet used by Databricks workspace
   - Public IP Address - Create a new, providing a human-readable name
   - Enable active-active mode - Enabled
   - Second Public IP Address - Create a new, providing a human-readable name
   - Configure BGP - Enabled
   - ASN - Default or any allowable value
   - Custom Azure APIPA BGP IP address - Input a valid value between 169.254.21.* and 169.254.22.*
   - Second Custom Azure APIPA BGP IP address - Input a valid value between 169.254.21.* and 169.254.22.*

 - Record the following values for future usage:
   - The ASN value (noted as AZURE_ASN below), 
   - Custom Azure APIPA BGP IP address (noted as AZURE_BGP_IP_0 below), Second Custom Azure APIPA BGP IP address (noted as AZURE_BGP_IP_1 below),
   - Public IP address (noted as AZURE_GW_IP_0 below),
   - Second public IP address (noted as AZURE_GW_IP_1 below)

 - In the GCP portal, create a VPN gateway with Network and Region configuration corresponding to Databricks workspace. Record the following values for future usage:
   - Public IP of interface0 (noted as HA_VPN_INT_0 below),
   - Public IP of interface1 (noted as HA_VPN_INT_1 below) 
 - In the GCP portal, create a Cloud Router with Network and Region configuration corresponding to Databricks workspace, and an ASN number. A valid ASN number must be within 64512-65534 or 4200000000-4294967294, and must not overlap with the above AZURE_ASN. Record the ASN number (noted as GOOGLE_ASN)
 - In the GCP portal, create a Peer VPN Gateway. Select 2 interfaces, and provide AZURE_GW_IP_0 and AZURE_GW_IP_1 as its 2 public IP addresses.
 - In the GCP portal, create 2 VPN tunnels with the following configuration:
   - VPN Gateway - Select the VPN Gateway created above
   - Peer VPN Gateway - Select the peer VPN Gateway created above
   - Cloud Router - Select the Cloud Route created above
   - Associated peer VPN gateway interface - Interface 0, 1 of Peer VPN Gateway, respectively
   - IKE version - IKEv2
   - IKE pre-shared key - Input or generate, and copy the value as SHARED_KEY_0, SHARED_KEY_1, respectively
   - BGP session config: peer ASN - AZURE_ASN
   - BGP session config: Cloud Router BGP IP address - Because the allowed ranges for Azure APIPA BGP peering IP addresses are 169.254.21.* and 169.254.22.*, you must select an available IP address in the /30 CIDR of those ranges
   - BGP session config: BGP peer IP address - AZURE_BGP_IP_0, AZURE_BGP_IP_1, respectively
 - In Azure portal, create 2 Local Network Gateways with the following configurations:
   - Endpoint - Select IP address
   - IP address - HA_VPN_INT_0 for first gateway, HA_VPN_INT_1 for second gateway
   - Address space - CIDR of GCP Databricks workspace subnet 
   - Configure BGP settings - Yes
   - ASN - GOOGLE_ASN
   - BGP peer IP address - GOOGLE_BGP_IP_0 for the first gateway, GOOGLE_BGP_IP_1 for the second gateway
 - In the Azure portal, navigate to the Virtual Network Gateway created above, click on the “Connections” tab, and create 2 VPN connections with the following configurations:
   - Connection type - Site-to-Site (IPsec)
   - Local Network Gateway - Select the first, and second local network gateway created above when creating the first and second connection, respectively
   - Shared Key (PSK) - SHARED_KEY_0 for the first connection creation, SHARED_KEY_1 for the second connection creation 
   - Enable BGP - Yes
   - IKE Protocol - IKEv2

 - With the BCG session we configured in the above step, Google Cloud Router will automatically learn routing rules and make them available to clusters in Google Cloud Subnet. Next we need to configure DNS to resolve Azure storage account FQDN to the private IP of Azure private endpoint:
 - Create a private DNS zone for “dfs.core.windows.net”, and attach it to the Google Cloud VPC
   - Add an A record, with “<storage account name>”  pointing to an IP address of the storage account private endpoint in Azure
 - Test connection:
    - Spin up a cluster in GCP workspace
    - Query the Azure shared table
    - The cluster should connect with the hosted zone associated with the VPC to resolve the name of the storage account to its private IP
  
