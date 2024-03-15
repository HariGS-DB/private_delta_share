# AWS (Producer) to GCP (Consumer) 
Here AWS Databricks acts as a data producer and Azure Databricks acts as the consumer. 

# Network Architecture:
![AWS-to-GCP.png](images%2FAWS-to-GCP.png)

# Private Storage Network:
The technique to protect data in the S3 bucket from public access is the same as we described in Section “AWS to Azure”. The things we need to pay attention to are as follows:
 - When defining bucket policy to restrict networks’ access.
 - The CIDR of the GCP workspace is not the subnet CIDR. As Databricks on GCP is built on GKE, where clusters are launched as pods in the GKE cluster, the CIDR used by clusters is the secondary IP range (see network requirements)
 - The Security Group used by the S3 VPC endpoint must allow traffic from the secondary IP range of the GCP workspace subnet.

# Cross-Cloud Connection Instruction:
 - create a site-2-site VPN connection between AWS and GCP. Refer to Google cloud documentation here if you have doubts.
  - In the GCP portal, create a VPN gateway with Network and Region configuration corresponding to Databricks workspace. Record the following values for future usage:
     - Public IP of interface0 (noted as INTERFACE_0_IP below),
     - Public IP of interface1 (noted as INTERFACE_1_IP below)
  - In the GCP portal, create a Cloud Router with Network and Region configuration corresponding to Databricks workspace, and an ASN number. A valid ASN number must be within 64512-65534 or 4200000000-4294967294. Record the ASN number (noted as GOOGLE_ASN)
  - In AWS portal, create 2 Customer Gateways with GOOGLE_ASN as BGP ASN, and INTERFACE_0_IP, INTERFACE_1_IP as IP address, respectively.
  - In the AWS portal, create a Virtual Private Gateway, with a valid number as a custom ASN. A valid ASN number must be within 64512-65534 or 4200000000-4294967294, and must not overlap with GOOGLE_ASN. Record this ASN number (notes as AWS_SIDE_ASN)
  - In the AWS portal, navigate to Site-to-Site VPN connections, and create 2 VPN connections with the following configurations:
    - Target gateway - Select the Virtual Private Gateway created above
    - Custom gateway - Select the first, and second customer gateways created above, when creating the first, and second VPN connections respectively
    - Routing options - Dynamic
  - Upon successful creation of 2 VPN connections, download the 2 configuration files. 
  - In the first file: note down:
    - pre-shared key of tunnel #1 as SHARED_SECRET_1, 
    - inside IP of virtual private gateway of tunnel #1 as AWS_T1_IP, 
    - inside IP of customer gateway of tunnel #1 as GCP_BGP_IP_TUNNEL_1,
    - Outside IP for the virtual private gateway of tunnel #1 as AWS_GW_IP_1,
    - pre-shared key of tunnel #2 as SHARED_SECRET_2, 
    - inside IP of virtual private gateway of tunnel #2 as AWS_T2_IP,
    - inside IP of customer gateway of tunnel #2 as GCP_BGP_IP_TUNNEL_2,
    - Outside IP for the virtual private gateway of tunnel #2 as AWS_GW_IP_2.
    - In the second file: note down:
    - pre-shared key of tunnel #1 as SHARED_SECRET_3, 
    - inside IP of virtual private gateway of tunnel #1 as AWS_T3_IP, 
    - inside IP of customer gateway of tunnel #1 as GCP_BGP_IP_TUNNEL_3,
    - Outside IP for the virtual private gateway of tunnel #1 as AWS_GW_IP_3,
    - pre-shared key of tunnel #2 as SHARED_SECRET_4, 
    - inside IP of virtual private gateway of tunnel #2 as AWS_T4_IP,
    - inside IP of customer gateway of tunnel #2 as GCP_BGP_IP_TUNNEL_4,
    - Outside IP for the virtual private gateway of tunnel #2 as AWS_GW_IP_4,
  - In the GCP portal, create a peer VPN gateway with 4 interfaces, and input above noted 4 outside IP addresses of the virtual private gateway as interface IP addresses.
  - In the GCP portal, create 4 VPN tunnels with the following configuration:
    - VPN Gateway - Select the VPN Gateway created above
    - Peer VPN Gateway - Select the peer VPN Gateway created above
    - Cloud Router - Select the Cloud Route created above
    - Associated peer VPN gateway interface - Interface 0, 1, 2, 3 of Peer VPN Gateway, respectively
    - IKE version - IKEv1
    - IKE pre-shared key - SHARED_SECRET_1, SHARED_SECRET_2, SHARED_SECRET_3, SHARED_SECRET_4, respectively
    - BGP session config: peer ASN - AWS_SIDE_ASN
    - BGP session config: Cloud Router BGP IP address - GCP_BGP_IP_TUNNEL_1, GCP_BGP_IP_TUNNEL_2, GCP_BGP_IP_TUNNEL_3, GCP_BGP_IP_TUNNEL_4, respectively
    - BGP session config: BGP peer IP address - AWS_T1_IP, AWS_T2_IP, AWS_T3_IP, AWS_T4_IP, respectively
  - You can quickly validate if the VPN tunnel status is “UP” by navigating to AWS console -> VPC -> Site-to-Site VPN connections -> your VPN connection -> Tunnel details tab
# Private DNS config:
 - To make clusters in the GCP workspace access the S3 bucket through the VPN tunnel and S3 VPC endpoint, we shall create a private DNS to resolve the S3 domain to the private IP of the S3 VPC endpoint (Interface type).
   - Create a private Cloud DNS in GCP, with “s3.<YOUR_BUCKET_REGION>.amazonaws.com” as the DNS name, adding the VPC of your GCP workspace as a network.
   - Then create an A record, resolving “<BUCKET_NAME>.s3.<REGION>.amazonaws.com” to the private IP of your interface type S3 VPC endpoint.
 - Route config: In AWS, add the following rule to the route table associated with the subnet where the S3 VPC endpoint is deployed:
   - Destination: secondary IP range of Subnet used by GCP workspace
   - Target: the Virtual Private Gateway used for VPN connection.
