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

In the case of a shared vpc: 

```shell
export SERVICE_NAME="private-cloudrun-app"
export CONNECTOR_NAME="shared-connector"
export VPC_NAME="shared-vpc"
export SUBNET_IP_RANGE="10.0.2.0/28"
export REGION="southamerica-west1"
export IMAGE_URL="southamerica-west1-docker.pkg.dev/dryruns/private-cloudrun-app/private-cloudrun-app:latest"
```

Change projects using

```shell
gcloud config set project dryruns
```

```shell
gcloud config set project my-shared-vpc-432520
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
gcloud compute networks vpc-access connectors create $CONNECTOR_NAME \
    --network $VPC_NAME \
    --range $SUBNET_IP_RANGE \
    --region $REGION
```

Expected result: 

```shell
Create request issued for: [my-connector]
Waiting for operation [projects/dryruns/locations/southamerica-west1/operations/45996389-ee09-4faa-9eac-cb339f475f4d] to complete...done.                                                                                                                                    
Created connector [my-connector].
```


Deploy Service

```shell
gcloud run deploy $SERVICE_NAME \
    --image $IMAGE_URL \
    --region $REGION \
    --vpc-connector $CONNECTOR_NAME \
    --ingress internal \
    --no-allow-unauthenticated
```

Expected result: 

```shell
Deploying container to Cloud Run service [private-cloudrun-app] in project [dryruns] region [southamerica-west1]
OK Deploying... Done.                                                                                                                                                                                                                                                        
  OK Creating Revision...                                                                                                                                                                                                                                                    
  OK Routing traffic...                                                                                                                                                                                                                                                      
  OK Setting IAM Policy...                                                                                                                                                                                                                                                   
Done.                                                                                                                                                                                                                                                                        
Service [private-cloudrun-app] revision [private-cloudrun-app-00002-7pp] has been deployed and is serving 100 percent of traffic.
Service URL: https://private-cloudrun-app-551624959543.southamerica-west1.run.app
```


After this, you should be able to consume the service only privately from the VPC where it was created, however notice that as `no-allow-unauthenticated` is set, then you either need to bind a service account or open it up for `allow-unauthenticated`


These are the options: 

1. Get Service account from consume: 

```shell
gcloud compute instances describe test-instance --zone southamerica-west1-a --format="get(serviceAccounts)"


{'email': '551624959543-compute@developer.gserviceaccount.com', 'scopes': ['https://www.googleapis.com/auth/devstorage.read_only', 'https://www.googleapis.com/auth/logging.write', 'https://www.googleapis.com/auth/monitoring.write', 'https://www.googleapis.com/auth/service.management.readonly', 'https://www.googleapis.com/auth/servicecontrol', 'https://www.googleapis.com/auth/trace.append']}
```


Based on this, perform policy binding

```shell
gcloud run services add-iam-policy-binding $SERVICE_NAME \
    --region REGION \
    --member "serviceAccount:551624959543-compute@developer.gserviceaccount.com" \
    --role roles/run.invoker

```


2. Open it up for unauthenticated consumption

```shell
gcloud run deploy $SERVICE_NAME \
    --image $IMAGE_URL \
    --region $REGION \
    --vpc-connector $CONNECTOR_NAME \
    --ingress internal \
    --allow-unauthenticated
```


---

# Put Load Balancer on top of the service

Export the following variables

```shell
export LOAD_BALANCER_NAME="private-cloudrun-lb"
export BACKEND_SERVICE_NAME="cloudrun-backend"
export NEG_NAME="cloudrun-neg"
export REGION="southamerica-west1"
export URL_MAP_NAME="cloudrun-url-map"
export HTTP_PROXY_NAME="cloudrun-http-proxy"
export FORWARDING_RULE_NAME="cloudrun-forwarding-rule"
export INTERNAL_IP_NAME="cloudrun-internal-ip"
export SUBNET_NAME="custom-subnet-santiago"
```


Create the NEG

```shell
gcloud compute network-endpoint-groups create $NEG_NAME \
    --region $REGION \
    --network-endpoint-type=serverless \
    --cloud-run-service $SERVICE_NAME \
    --cloud-run-region $REGION
```


Create the BES 

```shell
gcloud compute backend-services create $BACKEND_SERVICE_NAME \
    --load-balancing-scheme=INTERNAL_MANAGED \
    --protocol=HTTP \
    --region=$REGION \
    --health-checks="default"
```

Add the NEG to the BES 

```shell
gcloud compute backend-services add-backend $BACKEND_SERVICE_NAME \
    --network-endpoint-group=$NEG_NAME \
    --network-endpoint-group-region=$REGION \
    --region=$REGION
```

Reserve an internal IP Address

```shell
gcloud compute addresses create $INTERNAL_IP_NAME \
    --region=$REGION \
    --subnet=$SUBNET_NAME \
    --addresses-type=INTERNAL
```

Create URL Mapping

```shell
gcloud compute url-maps create $URL_MAP_NAME \
    --default-service=$BACKEND_SERVICE_NAME
```

Create HTTP Proxy Load Balancer

```shell
gcloud compute target-http-proxies create $HTTP_PROXY_NAME \
    --url-map=$URL_MAP_NAME
```


Create Forwarding Rule

```shell
gcloud compute forwarding-rules create $FORWARDING_RULE_NAME \
    --load-balancing-scheme=INTERNAL_MANAGED \
    --address=$(gcloud compute addresses describe $INTERNAL_IP_NAME --region $REGION --format="get(address)") \
    --region=$REGION \
    --network=$VPC_NAME \
    --target-http-proxy=$HTTP_PROXY_NAME \
    --ports=80
```