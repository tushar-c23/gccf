## Creating a Virtual Machine
### Set the region and zone
`gcloud config set compute/region REGION`<br>
`export REGION=REGION`<br>
`export ZONE=Zone`<br>

### Install NGINX web server in the VM instance
`sudo apt-get update`<br>
`sudo apt-get install nginx -y`<br>
`ps auwx | grep nginx` - To confirm if the NGINX web server is running<br>

### Create a new instance with gcloud
`gcloud compute instances create gcelab2 --machine-type e2-medium --zone=$ZONE`<br>
To use ssh to connect to the instance<br>
`gcloud compute ssh gcelab2 --zone=$ZONE`<br>

## Cloud Shell and Gcloud
Set region to \<REGION\><br>
`gcloud config set compute/region REGION`<br>
`gcloud config get-value compute/region`<br>
Set zone to \<ZONE\><br>
`gcloud config set compute/zone ZONE`<br>
`gcloud config get-value compute/zone`<br>

View details of the project<br>
`gcloud compute project-info describe --project $(gcloud config get-value project)`<br>

Setting env vars<br>
`export PROJECT_ID=$(gcloud config get-value project)`<br>
`export ZONE=$(gcloud config get-value compute/zone)`<br>

List the instances with filter<br>
`gcloud compute instances list --filter="name=('gcelab2')"`<br>

List firewall rules with filter<br>
`gcloud compute firewall-rules list`<br>
`gcloud compute firewall-rules list --filter="network='default'"`<br>
`gcloud compute firewall-rules list --filter="NETWORK:'default' AND ALLOW:'icmp'"`<br>

Connect to vm instance<br>
`gcloud compute ssh gcelab2 --zone $ZONE`<br>

### Updating the firewall
Add a tag to the vm<br>
`gcloud compute instances add-tags gcelab2 --tags http-server,https-server`<br>
`gcloud compute firewall-rules create default-allow-http --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server`<br>

Verify <br>
`curl http://$(gcloud compute instances list --filter=name:gcelab2 --format='value(EXTERNAL_IP)')`<br>

## Kubernetees Engine
### Create a GKE cluster
Create a cluster<br>
`gcloud container clusters create --machine-type=e2-medium --zone=ZONE lab-cluster`<br>
Get credentials for the cluster<br>
`gcloud container clusters get-credentials lab-cluster`<br>

### Deploying an application to the cluster
Create a deployment<br>
`kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0`<br>
Create a kubernetes service<br>
`kubectl expose deployment hello-server --type=LoadBalancer --port 8080`<br>
Inspect<br>
`kubectl get service`<br>

### Deleting a cluster
`gcloud container clusters delete lab-cluster`<br>

## Set Up Network and HTTP Load Balancers
### Create multiple web server instances
```bash
gcloud compute instances create www1 \
    --zone=Zone \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www1</h3>" | tee /var/www/html/index.html'
```
Create a firewall rule<br>
```bash
gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags network-lb-tag --allow tcp:80
```
### Configure load balancing
Create a static external IP address for your load balancer:
```bash
gcloud compute addresses create network-lb-ip-1 \
  --region Region
```
Add a legacy HTTP health check resource:
```bash
gcloud compute http-health-checks create basic-check
```
Add a target pool in the same region as your instances. Run the following to create the target pool and use the health check, which is required for the service to function:
```bash
gcloud compute target-pools create www-pool \
  --region Region --http-health-check basic-check
```
Add your instances to the target pool:
```bash
gcloud compute target-pools add-instances www-pool \
    --instances www1,www2,www3
```
Add a forwarding rule
```bash
gcloud compute forwarding-rules create www-rule \
    --region  Region \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool
```

### Sending traffic to your instances
Enter the following command to view the external IP address of the www-rule forwarding rule used by the load balancer:<br>
```bash
gcloud compute forwarding-rules describe www-rule --region Region
```
Access the external IP address
```bash
IPADDRESS=$(gcloud compute forwarding-rules describe www-rule --region Region --format="json" | jq -r .IPAddress)
```

## Create an HTTP load balancer
### First, create the load balancer template:
```bash
gcloud compute instance-templates create lb-backend-template \
   --region=Region \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=debian-11 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'
```
Create a managed instance group based on the template:
```bash
gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template --size=2 --zone=Zone
```
Create the fw-allow-health-check firewall rule.
```bash
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80
```
Now that the instances are up and running, set up a global static external IP address that your customers use to reach your load balancer:
```bash
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global
```
```bash
gcloud compute addresses describe lb-ipv4-1 \
  --format="get(address)" \
  --global
```
Create a health check for the load balancer:

```bash
gcloud compute health-checks create http http-basic-check \
  --port 80
```

Create a be service
```bash
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global
```

Add your instance group as the backend to the backend service:
```bash
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=Zone \
  --global
```

Create a URL map to route the incoming requests to the default backend service:
```bash
gcloud compute url-maps create web-map-http \
    --default-service web-backend-service
```

Create a target HTTP proxy to route requests to your URL map:

```bash
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map-http
```

Create a global forwarding rule to route incoming requests to the proxy:
```bash
gcloud compute forwarding-rules create http-content-rule \
   --address=lb-ipv4-1\
   --global \
   --target-http-proxy=http-lb-proxy \
   --ports=80
```