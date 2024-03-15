# AWS (Producer) to Azure (Consumer) 
Here AWS Databricks acts as a data producer and Azure Databricks acts as the consumer. 

# Network Architecture:
![aws-to-azure.png](images%2FAws-to-Azure.png)

# Private Storage Network:
 - Ensure S3 bucket is set to block public access
 - In addition to the metastore bucket policy (here), add policy permission to restrict which networks can connect to the s3 bucket (Reason here):
   - CIDR of AWS workspace VPC (Ep. 10.10.0.0/16)
   - CIDR of Azure workspace Vnet (Ep. 10.20.0.0/16)
   - Public IP of control plane NAT in your region (check here)
   - Public CIDR of Databricks standby infrastructure in your region (check here)
   - Further IPs which need to be whitelisted in your corporate (check here as a reference: link)
     Note: do remember to whitelist your corporate IP, otherwise you may lose the permission to configure this S3 bucket once you update the policy.
 - Create a subnet to host the endpoints:
   - Create an Interface type of S3 VPC endpoint (see doc), since Gateway type Endpoint is not reachable from other networks (no private IP address). For the security group, use the default one or create a new one. It must allow traffic from Azure Vnet CIDR.
   - As a recommendation, we also create a Gateway type of S3 VPC Endpoint (see doc). Edit the Interface type of the S3 VPC endpoint, and enable “private DNS only for inbound endpoints”. In this way, the traffic from VPC will go through the Gateway type endpoint (which is not billed), while inbound traffic from Azure Vnet through VPN will go to the Interface type endpoint. (see link)

# Cross-Cloud Connection Instruction:
 - Create Site-2-Site VPN connection between AWS VPC and Azure Vnet as described in “cross-cloud connection setup” in the above Section “Azure to AWS”, proceeding from step Nr.1 to Nr.15.
 - Private DNS config: To enable Azure Databricks clusters to resolve S3 FQDN to the private IP address of S3 VPC Endpoint within AWS workspace VPC.
   - Create a private DNS zone with the name “s3.<REGION>.amazonaws.com” in Azure console, replacing “<REGION>” with the region where AWS workspace is deployed. Ep. “eu-central-1”
   - In the new private DNS zone, create an “A” record set: With the S3 bucket name of AWS workspace metastore as “Name”, and the private IP address of S3 VPC endpoint (interface type) as IP.
 - Routing config:
In AWS, add following rule to the route table associated with the subnet where S3 VPC endpoint is deployed:
   - Destination: CIDR of Azure workspace Vnet
   - Target: select the Virtual Private Gateway used for VPN connection.
 - Test connection:
   - Spin up a cluster in Azure workspace
   - Query the AWS shared table
   - The cluster should connect with the DNS zone associated with the Vnet to resolve the name of the S3 bucket to its private IP
   - The cluster should be able to access the storage account through the routes configured from subnet route → NAT gateway route → virtual private gateway → tunnel → local network gateway → storage account

# Additional S3 bucket Policy:
When setting up AWS workspace, we configured “Block public access” on our metastore S3 bucket, which will deny incoming unauthorised access to data files.Instead of blocking any network traffic from the internet, this restriction is on the S3 service level, which does not block network traffic. Delta Sharing will create a pre-signed URL to allow recipients to temporarily access data files, hence we can still successfully query data files from Azure workspace with “Block public access” enabled. To restrict access based on source IP, we shall use bucket policy (example here)
