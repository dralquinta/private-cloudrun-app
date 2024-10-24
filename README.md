# private-cloudrun-app



Export the following variables: 

```shell
export SERVICE_NAME="private-cloudrun-app"
export CONNECTOR_NAME="my-connector"
export VPC_NAME="my-custom-vpc"
export SUBNET_IP_RANGE="10.0.0.128/28"
export REGION="southamerica-west1"
export IMAGE_URL="southamerica-west1-docker.pkg.dev/dryruns/private-cloudrun-app/private-cloudrun-app:latest"
```

Enable API:

```shell
gcloud services enable vpcaccess.googleapis.com
```

Make sure user has the following access: 

- Serverless VPC Access Admin
- Compute Network Admin

Create VPC Connector 

```shell
gcloud compute networks vpc-access connectors create CONNECTOR_NAME \
    --network NETWORK_NAME \
    --range SUBNET_IP_RANGE \
    --region REGION
```