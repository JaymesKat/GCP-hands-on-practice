# GCP-hands-on-practice

Documentation of completed GCP Qwiks labs for Google Africa Developer Associate Scholarship (GADS) program

**Completed Hands-on Labs** ([View screenshots here](/screenshots/list.md))

1. Cloud IAM
2. Working with multiple VPC networks
3. Controlling access to multiple VPC networks
4. Virtual Private Networks (VPN)
5. Controlling an HTTP Load Balancer with Autoscaling
6. Caching Storage Content with Cloud CDN
7. Implement Private Google Access and Cloud NAT
8. AK8S-03 Creating a GKE Cluster
9. Configuring an Internal Load Balancer
10. AK8S-05 Upgrading Kubernetes Engine Clusters
11. AK8S-06 Creating Kubernetes Engine Deployments

**Labs translated to gcloud command line instructions**

**Lab1: Configuring an Internal Load Balancer**

**Reference architecture**

![`image`](internal-load-balancer-diagram.png)

**Initial set up**

- Open command line terminal (Ensure you have [installed](https://cloud.google.com/sdk/install) and [set up](https://cloud.google.com/sdk/docs/initializing) Google Cloud SDK in your local environment)
- Set default project `gcloud config set project [PROJECT ID]` It will be referenced later using the `$DEV_SHELL_PROJECT_ID` variable.
- The network **my-internal-app** with **subnet-a** and **subnet-b** and firewall rules for **RDP**, **SSH**, and **ICMP** traffic have been configured for you.

**Steps**

1. **Create internal traffic and health check firewall rules**

- Create a firewall rule to allow traffic from any sources in the 10.10.0.0/16 range

  ```
  gcloud compute firewall-rules create fw-allow-lb-access \
        --direction=INGRESS \
        --priority=1000 \
        --network=my-internal-app \
        --action=ALLOW \
        --rules=all \
        --source-ranges=10.10.0.0/16 \
        --target-tags=backend-service
  ```

- Create the health check rule

  ```
  gcloud compute firewall-rules create fw-allow-health-checks \
  							--direction=INGRESS \
  							--priority=1000 \
  							--network=my-internal-app \
  							--action=ALLOW \
  							--rules=tcp:80 \
  							--source-ranges=130.211.0.0/22,35.191.0.0/16 \
  							--target-tags=backend-service
  ```

2. **Create a NAT configuration using Cloud Router**

- Create cloud router **nat-router-us-central1**

  ```
  gcloud compute routers create nat-router-us-central1 \
      --network my-internal-app \
      --region us-central1
  ```

- Configure the NAT gateway **nat-config**

  ```
  gcloud compute routers nats create nat-config \
      --router nat-router-us-central1 \
      --region us-central1 \
      --auto-allocate-nat-external-ips \
      --nat-all-subnet-ip-ranges
  ```

3. **Configure two instance templates**

- Configure first instance template

  ```
  gcloud beta compute instance-templates create instance-template-1 \
  		--project=$DEVSHELL_PROJECT_ID \
  		--machine-type=f1-micro \
  		--subnet=projects/$DEVSHELL_PROJECT_ID/regions/us-central1/subnetworks/subnet-a \
  		--no-address \
  		--metadata=startup-script-url=gs://cloud-training/gcpnet/ilb/startup.sh \
  		--maintenance-policy=MIGRATE \
  		--service-account=215988533831-compute@developer.gserviceaccount.com \
  		--scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append \
  		--region=us-central1 \
  		--tags=backend-service \
  		--image=debian-10-buster-v20200910 \
  		--image-project=debian-cloud \
  		--boot-disk-size=10GB \
  		--boot-disk-type=pd-standard \
  		--boot-disk-device-name=instance-template-1 \
  		--no-shielded-secure-boot \
  		--no-shielded-vtpm \
  		--no-shielded-integrity-monitoring \
  		--reservation-affinity=any
  ```

- Configure second instance template (Copy first instance template command and specify subnet-b as the subnet)

  ```
  gcloud beta compute instance-templates create instance-template-2 \
  		--project=$DEVSHELL_PROJECT_ID \
  		--machine-type=f1-micro \
  		--subnet=projects/$DEVSHELL_PROJECT_ID/regions/us-central1/subnetworks/subnet-b \
  		--no-address \
  		--metadata=startup-script-url=gs://cloud-training/gcpnet/ilb/startup.sh \
  		--maintenance-policy=MIGRATE \
  		--service-account=215988533831-compute@developer.gserviceaccount.com \
  		--scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append \
  		--region=us-central1 \
  		--tags=backend-service \
  		--image=debian-10-buster-v20200910 \
  		--image-project=debian-cloud \
  		--boot-disk-size=10GB \
  		--boot-disk-type=pd-standard \
  		--boot-disk-device-name=instance-template-2 \
  		--no-shielded-secure-boot \
  		--no-shielded-vtpm \
  		--no-shielded-integrity-monitoring \
  		--reservation-affinity=any
  ```

4. **Create two managed instance groups**

- Create first instance group based on first instance template 'instance-template-1'

  ```
  gcloud compute instance-groups managed create instance-group-1 \
  	 --project=$DEVSHELL_PROJECT_ID \
     --base-instance-name=instance-group-1 \
     --template=instance-template-1 \
     --size=1 \
     --zone=us-central1-a
  ```

- Configure auto-scaling for first instance group

  ```
  gcloud beta compute instance-groups managed set-autoscaling "instance-group-1" \
  	 --project=$DEVSHELL_PROJECT_ID \
     --zone "us-central1-a" \
     --cool-down-period "45" \
     --max-num-replicas "5" \
     --min-num-replicas "1" \
     --target-cpu-utilization "0.8" \
     --mode "on"
  ```

- Create a second instance group with autoscaling called 'instance-group-1' with the same fields as the first except template and zone i.e. instance-template-2 and us-central1-b respectively

  ```
  gcloud compute  instance-groups managed create instance-group-2 \
     --project=$DEVSHELL_PROJECT_ID \
     --base-instance-name=instance-group-2 \
     --template=instance-template-2 \
     --size=1 \
   --zone=us-central1-b

  gcloud beta compute instance-groups managed set-autoscaling "instance-group-2" \
  	 --project=$DEVSHELL_PROJECT_ID \
     --zone "us-central1-b" \
     --cool-down-period "45" \
     --max-num-replicas "5" \
     --min-num-replicas "1" \
     --target-cpu-utilization "0.8" \
     --mode "on"
  ```

5. **Configure an internal load balancer**

- Create Load Balancer Health check

  ```
  gcloud compute health-checks create tcp my-ilb-health-check \
      --region=us-central1 \
      --port=80
  ```

- Create Load Balancer Backend service

  ```
  gcloud compute backend-services create my-ilb \
      --load-balancing-scheme=internal \
      --protocol=tcp \
      --region=us-central1 \
      --health-checks=my-ilb-health-check \
      --health-checks-region=us-central1
  ```

- Attach the two instance groups to backend service

  ```
  gcloud compute backend-services add-backend my-ilb \
      --region=us-central1 \
      --instance-group=instance-group-1 \
      --instance-group-zone=us-central1-a
  gcloud compute backend-services add-backend my-ilb \
      --region=us-central1 \
      --instance-group=instance-group-2 \
      --instance-group-zone=us-central1-b
  ```

- Configure frontend, set forwarding rule for Load Balancer

  ```
  gcloud compute forwarding-rules create fr-my-ilb \
      --region=us-central1 \
      --load-balancing-scheme=internal \
      --network=my-internal-app \
      --subnet=subnet-b \
      --address=10.10.30.5 \
      --ip-protocol=TCP \
      --ports=80 \
      --backend-service=be-my-ilb \
      --backend-service-region=us-central1
  ```

**Lab 2: AK8S-05 Upgrading Kubernetes Engine Clusters**

​ **Initial Setup**
Set default project `gcloud config set project [PROJECT ID]` It will be referenced later using the `$DEV_SHELL_PROJECT_ID` variable.

**Steps**

1. **Deploy a GKE cluster**

```
gcloud beta container --project $DEVSHELL_PROJECT_ID clusters create "cluster-1" --zone "us-central1-c" --no-enable-basic-auth --cluster-version "1.15.12-gke.2" --machine-type "e2-medium" --image-type "COS" --disk-type "pd-standard" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "3" --enable-stackdriver-kubernetes --enable-ip-alias --network "projects/$DEVSHELL_PROJECT_ID/global/networks/default" --subnetwork "projects/$DEVSHELL_PROJECT_ID/regions/us-central1/subnetworks/default" --default-max-pods-per-node "110" --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0
```

2. **Upgrade your GKE cluster**

Upgrade your cluster master version

```
gcloud container clusters upgrade cluster-1 \
  --zone "us-central1-c" \
  --master --cluster-version 1.16.13-gke.400
```

Upgrade node pool version

```
gcloud container clusters upgrade cluster-1 \
  --zone "us-central1-c" \
  --node-pool=default-pool \
  --cluster-version 1.16.13-gke.400
```
