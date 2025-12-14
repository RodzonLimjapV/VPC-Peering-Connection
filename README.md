# VPC Peering: RDS PostgreSQL (AWS Account-A) ↔ MS Fabric (AWS Account-B)

This repository documents how to connect an **RDS PostgreSQL** database
in the **MySolar AWS account** to **Microsoft Fabric** in the **BSI AWS
account**, using **VPC Peering**.

Connectivity path:

**MS Fabric → Data Gateway EC2 (BSI VPC) → VPC Peering → RDS PostgreSQL
(MySolar VPC)**

------------------------------------------------------------------------

## 1. Architecture Overview

> 🖼️ <img width="1291" height="886" alt="image" src="https://github.com/user-attachments/assets/5d84c62d-0763-437a-9bbd-2714d32109ac" />


**Account A --- MySolar** - RDS PostgreSQL (`rts-dev`, `rts-qa`,
`rts-prod`) - Private RDS subnet group\
- VPC CIDR (example): `10.10.0.0/16`

**Account B --- BSI** - Microsoft Fabric\
- Data Gateway EC2 host\
- VPC CIDR (example): `10.20.0.0/16`

**Connectivity** - VPC Peering (bidirectional routes) - DNS resolution
enabled over peering - Security groups restrict port `5432`

------------------------------------------------------------------------

## 2. Prerequisites

### CLI Profiles Setup (Windows CMD)

``` cmd
aws configure --profile mysolar
aws configure --profile bsi
```

Environment variables (example):

``` cmd
set MYSOLAR_PROFILE=mysolar
set MYSOLAR_REGION=ap-southeast-1
set MYSOLAR_VPC_ID=vpc-aaaaaaaa
set MYSOLAR_VPC_CIDR=10.10.0.0/16

set BSI_PROFILE=bsi
set BSI_REGION=ap-southeast-1
set BSI_VPC_ID=vpc-bbbbbbbb
set BSI_VPC_CIDR=10.20.0.0/16

set RDS_SG_ID=sg-rds-mysolar
set BSI_DG_EC2_SG_ID=sg-datagateway-bsi
```

------------------------------------------------------------------------

## 3. Create VPC Peering Connection (CLI / CMD)

### Step 3.1 --- Create Peering Request from MySolar → BSI

``` cmd
aws ec2 create-vpc-peering-connection ^
  --profile %MYSOLAR_PROFILE% ^
  --region %MYSOLAR_REGION% ^
  --vpc-id %MYSOLAR_VPC_ID% ^
  --peer-vpc-id %BSI_VPC_ID% ^
  --peer-owner-id <BSI_ACCOUNT_ID> ^
  --tag-specifications "ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=peering-mysolar-bsi-rds}]" ^
  --query "VpcPeeringConnection.VpcPeeringConnectionId" ^
  --output text
```

Set the returned ID:

``` cmd
set VPC_PEERING_ID=pcx-xxxxxxxxxxxxxxxxx
```

------------------------------------------------------------------------

### Step 3.2 --- Accept Peering in BSI Account

``` cmd
aws ec2 accept-vpc-peering-connection ^
  --profile %BSI_PROFILE% ^
  --region %BSI_REGION% ^
  --vpc-peering-connection-id %VPC_PEERING_ID%
```

------------------------------------------------------------------------

### Step 3.3 --- Enable DNS Resolution Over Peering

``` cmd
aws ec2 modify-vpc-peering-connection-options ^
  --profile %MYSOLAR_PROFILE% ^
  --region %MYSOLAR_REGION% ^
  --vpc-peering-connection-id %VPC_PEERING_ID% ^
  --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true

aws ec2 modify-vpc-peering-connection-options ^
  --profile %BSI_PROFILE% ^
  --region %BSI_REGION% ^
  --vpc-peering-connection-id %VPC_PEERING_ID% ^
  --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true
```

------------------------------------------------------------------------

## 4. Update Route Tables

### MySolar → Add Route to BSI VPC CIDR

``` cmd
aws ec2 create-route ^
  --profile %MYSOLAR_PROFILE% ^
  --region %MYSOLAR_REGION% ^
  --route-table-id <MYSOLAR_RT_ID> ^
  --destination-cidr-block %BSI_VPC_CIDR% ^
  --vpc-peering-connection-id %VPC_PEERING_ID%
```

### BSI → Add Route to MySolar VPC CIDR

``` cmd
aws ec2 create-route ^
  --profile %BSI_PROFILE% ^
  --region %BSI_REGION% ^
  --route-table-id <BSI_RT_ID> ^
  --destination-cidr-block %MYSOLAR_VPC_CIDR% ^
  --vpc-peering-connection-id %VPC_PEERING_ID%
```

------------------------------------------------------------------------

## 5. Security Group Configuration

### Allow BSI Data Gateway to Access MySolar RDS

``` cmd
aws ec2 authorize-security-group-ingress ^
  --profile %MYSOLAR_PROFILE% ^
  --region %MYSOLAR_REGION% ^
  --group-id %RDS_SG_ID% ^
  --ip-permissions IpProtocol=tcp,FromPort=5432,ToPort=5432,UserIdGroupPairs=[{GroupId=%BSI_DG_EC2_SG_ID%}]
```

------------------------------------------------------------------------

## 6. Connectivity Test

### Windows PowerShell

``` powershell
Test-NetConnection <RDS-ENDPOINT> -Port 5432
```

### Linux

``` bash
nc -vz <RDS-ENDPOINT> 5432
```

------------------------------------------------------------------------

## 7. Key Benefits of Using VPC Peering

1.  **Private & Secure (No Public Access)**
2.  **Low Latency & High Bandwidth**
3.  **Zero Data Transfer Cost (Same Region)**
4.  **Simple Architecture**
5.  **Cross-Account Support**
6.  **DNS Resolution for RDS**
7.  **Strong Security Model**
8.  **Reliable for Long-Term Integrations**

------------------------------------------------------------------------

## 8. Common Use Cases

1.  **Cross-Account Database Access**
2.  **Centralized Analytics**
3.  **Shared Services Architecture**
4.  **Multi-Account Microservices**
5.  **Migration Workloads**
6.  **VPC-to-VPC API Integration**
7.  **Multi-Business Unit Connectivity**

------------------------------------------------------------------------

## 9. Cleanup

``` cmd
aws ec2 delete-vpc-peering-connection ^
  --profile %MYSOLAR_PROFILE% ^
  --region %MYSOLAR_REGION% ^
  --vpc-peering-connection-id %VPC_PEERING_ID%
```

------------------------------------------------------------------------
