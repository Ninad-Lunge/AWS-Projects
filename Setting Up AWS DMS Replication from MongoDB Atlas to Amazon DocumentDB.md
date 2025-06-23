# Runbook: Setting Up AWS DMS Replication from MongoDB Atlas to Amazon DocumentDB

## 1. Prerequisites

### 1.1 AWS Account Setup
• Ensure you have an AWS account with appropriate permissions
• Install and configure AWS CLI with a profile
• Ensure you have access to both MongoDB Atlas and Amazon DocumentDB

### 1.2 Network Configuration
• Ensure VPC peering or appropriate network connectivity between MongoDB Atlas and AWS
• For MongoDB Atlas, whitelist the CIDR range of your AWS VPC

## 2. Create Required AWS Resources

### 2.1 Create VPC Security Groups
bash
# Create security group for DocumentDB
aws ec2 create-security-group \
  --group-name documentdb-sg \
  --description "Security group for DocumentDB" \
  --vpc-id <your-vpc-id> \
  --profile <your-profile> \
  --region <your-region>

# Create security group for DMS
aws ec2 create-security-group \
  --group-name dms-sg \
  --description "Security group for DMS" \
  --vpc-id <your-vpc-id> \
  --profile <your-profile> \
  --region <your-region>

# Add inbound rule to DocumentDB security group to allow traffic from DMS security group
aws ec2 authorize-security-group-ingress \
  --group-id <documentdb-sg-id> \
  --protocol tcp \
  --port 27017 \
  --source-group <dms-sg-id> \
  --profile <your-profile> \
  --region <your-region>

# Add outbound rule to DMS security group (allow all outbound traffic)
aws ec2 authorize-security-group-egress \
  --group-id <dms-sg-id> \
  --protocol -1 \
  --port -1 \
  --cidr 0.0.0.0/0 \
  --profile <your-profile> \
  --region <your-region>


### 2.2 Create Subnet Groups
bash
# Create subnet group for DocumentDB
aws docdb create-db-subnet-group \
  --db-subnet-group-name documentdb-subnet-group \
  --db-subnet-group-description "Subnet group for DocumentDB" \
  --subnet-ids <subnet-id-1> <subnet-id-2> <subnet-id-3> \
  --profile <your-profile> \
  --region <your-region>

# Create subnet group for DMS (using the same subnets as DocumentDB)
aws dms create-replication-subnet-group \
  --replication-subnet-group-identifier dms-subnet-group \
  --replication-subnet-group-description "Subnet group for DMS" \
  --subnet-ids <subnet-id-1> <subnet-id-2> <subnet-id-3> \
  --profile <your-profile> \
  --region <your-region>


### 2.3 Create DocumentDB Cluster
bash
# Create parameter group for DocumentDB
aws docdb create-db-cluster-parameter-group \
  --db-cluster-parameter-group-name docdb-param-group \
  --db-parameter-group-family docdb5.0 \
  --description "Parameter group for DocumentDB" \
  --profile <your-profile> \
  --region <your-region>

# Modify parameter group to disable TLS (if needed)
aws docdb modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name docdb-param-group \
  --parameters "ParameterName=tls,ParameterValue=disabled,ApplyMethod=pending-reboot" \
  --profile <your-profile> \
  --region <your-region>

# Create DocumentDB cluster
aws docdb create-db-cluster \
  --db-cluster-identifier <your-docdb-cluster-name> \
  --engine docdb \
  --master-username <admin-username> \
  --master-user-password <admin-password> \
  --db-subnet-group-name documentdb-subnet-group \
  --vpc-security-group-ids <documentdb-sg-id> \
  --db-cluster-parameter-group-name docdb-param-group \
  --profile <your-profile> \
  --region <your-region>

# Create DocumentDB instance
aws docdb create-db-instance \
  --db-instance-identifier <your-docdb-instance-name> \
  --db-instance-class db.r5.large \
  --engine docdb \
  --db-cluster-identifier <your-docdb-cluster-name> \
  --profile <your-profile> \
  --region <your-region>


## 3. Set Up DMS Endpoints

### 3.1 Create MongoDB Atlas Source Endpoint
bash
aws dms create-endpoint \
  --endpoint-identifier mongodb-atlas-source-ep \
  --endpoint-type source \
  --engine-name mongodb \
  --username <mongodb-username> \
  --password <mongodb-password> \
  --server-name <mongodb-atlas-hostname> \
  --port 27017 \
  --mongodb-settings "AuthType=PASSWORD,AuthMechanism=SCRAM-SHA-1,AuthSource=admin,NestingLevel=NONE,ExtractDocId=false" \
  --profile <your-profile> \
  --region <your-region>


### 3.2 Create DocumentDB Target Endpoint
bash
aws dms create-endpoint \
  --endpoint-identifier documentdb-target-ep \
  --endpoint-type target \
  --engine-name docdb \
  --username <docdb-username> \
  --password <docdb-password> \
  --server-name <docdb-cluster-endpoint> \
  --port 27017 \
  --profile <your-profile> \
  --region <your-region>


## 4. Set Up DMS Serverless Replication

### 4.1 Create DMS Replication Configuration
bash
aws dms create-replication-config \
  --replication-config-identifier <your-replication-name> \
  --source-endpoint-arn <mongodb-source-endpoint-arn> \
  --target-endpoint-arn <documentdb-target-endpoint-arn> \
  --replication-type full-load \
  --compute-config '{"MinCapacityUnits": 16, "MaxCapacityUnits": 32, "ReplicationSubnetGroupId": "dms-subnet-group", "VpcSecurityGroupIds": ["<dms-sg-id>"]}' \
  --table-mappings '{"rules":[{"rule-id":"1","rule-name":"1","rule-type":"selection","rule-action":"include","object-locator":{"schema-name":"%","table-name":"%"},"filters":[]}]}' \
  --profile <your-profile> \
  --region <your-region>


### 4.2 Start DMS Replication
bash
aws dms start-replication \
  --replication-config-arn <replication-config-arn> \
  --start-replication-type reload-target \
  --profile <your-profile> \
  --region <your-region>


## 5. Monitor Replication Progress

### 5.1 Check Replication Status
bash
aws dms describe-replications \
  --filters Name=replication-config-arn,Values=<replication-config-arn> \
  --profile <your-profile> \
  --region <your-region>


### 5.2 Check Table Statistics
bash
aws dms describe-table-statistics \
  --replication-config-arn <replication-config-arn> \
  --profile <your-profile> \
  --region <your-region>


## 6. Troubleshooting Common Issues

### 6.1 Connection Issues
• **Security Group Configuration**: Ensure DocumentDB security group allows traffic from DMS security group on port
27017
• **Subnet Configuration**: Ensure DMS and DocumentDB subnet groups share the same subnets
• **Network Connectivity**: Verify network connectivity between DMS and both endpoints

### 6.2 TLS/SSL Issues
• **TLS Configuration**: If DocumentDB has TLS enabled, but DMS endpoint doesn't use SSL:

bash
  # Disable TLS on DocumentDB (temporary solution)
  aws docdb create-db-cluster-parameter-group \
    --db-cluster-parameter-group-name docdb-no-tls \
    --db-parameter-group-family docdb5.0 \
    --description "DocumentDB parameter group with TLS disabled" \
    --profile <your-profile> \
    --region <your-region>

  aws docdb modify-db-cluster-parameter-group \
    --db-cluster-parameter-group-name docdb-no-tls \
    --parameters "ParameterName=tls,ParameterValue=disabled,ApplyMethod=pending-reboot" \
    --profile <your-profile> \
    --region <your-region>

  aws docdb modify-db-cluster \
    --db-cluster-identifier <your-docdb-cluster-name> \
    --db-cluster-parameter-group-name docdb-no-tls \
    --apply-immediately \
    --profile <your-profile> \
    --region <your-region>

  # Reboot the primary instance to apply changes
  aws docdb reboot-db-instance \
    --db-instance-identifier <primary-instance-id> \
    --profile <your-profile> \
    --region <your-region>



• **Configure SSL on DMS endpoint**:

bash
  aws dms modify-endpoint \
    --endpoint-arn <documentdb-target-endpoint-arn> \
    --extra-connection-attributes "ssl=true" \
    --profile <your-profile> \
    --region <your-region>



### 6.3 Authentication Issues
• Verify MongoDB Atlas username and password
• Ensure MongoDB Atlas user has appropriate permissions
• Verify DocumentDB username and password
• Check if MongoDB Atlas requires specific authentication mechanisms

### 6.4 Performance Issues
• Increase DMS capacity units if replication is slow
• Check for network latency between MongoDB Atlas and AWS
• Monitor CPU and memory usage on DocumentDB instances

## 7. Best Practices

1. Test in a Non-Production Environment First: Always test the replication in a non-production environment before
implementing in production.

2. Monitor Replication Logs: Enable CloudWatch logging for DMS to troubleshoot issues.

3. Regular Backups: Take regular backups of your DocumentDB cluster before and during the migration.

4. Validate Data: After replication, validate that all data has been correctly migrated.

5. Security: Re-enable TLS after successful migration for enhanced security.

6. Performance Tuning: Adjust DMS capacity units and DocumentDB instance sizes based on your workload.

7. Network Optimization: Ensure low latency between MongoDB Atlas and AWS for better performance.

## 8. Post-Migration Steps

1. Verify Data Integrity: Compare record counts and sample data between source and target.

2. Update Application Connection Strings: Update your applications to connect to DocumentDB.

3. Monitor Performance: Monitor DocumentDB performance to ensure it meets your requirements.

4. Clean Up Resources: Delete unnecessary DMS resources to avoid ongoing charges.

5. Re-enable Security Features: Re-enable TLS and other security features that were disabled during migration.
