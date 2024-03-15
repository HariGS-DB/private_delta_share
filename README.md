# Private Delta Sharing
This Repo contains instructions on how to create private delta share cross-cloud

# Prerequisites:

Before we go into the details of the secure delta share architecture, it's important to list the prerequisite with respect to Databricks workspace/account set-up and the cloud infra that should be in place.
# Databricks:
 - UC metastore created in the respective cloud account console with delta sharing enabled
 - Databricks workspace set up on two clouds (aws-azure, aws-gcp, gcp-azure) and mapped to the metastore
 - D2D delta share set up between two clouds workspaces:
 - Add consumer DB id as a recipient
 - Create a share
 - Add tables to the share
 - Grant access on that share to the recipient
# Cloud Infra:
 - Customer VNet set up
 - Data stored in customer storage (s3, adls, gcs) 
 - Securing the cloud storage (explained below in the respective section)
 - Cloud permission to create the necessary infrastructure for secure VPN gateway connectivity
# Cross Cloud Instruction:
Please refer to instruction below for cloud-specific private delta share infrastructure instruction.
