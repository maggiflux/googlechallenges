# GCPChallengeAssociate
Google cloud challenges solutions

# **Implement Load Balancing on Compute Engine: Challenge Lab**

<aside>

# **Challenge scenario**

You have started a new role as a Junior Cloud Engineer for Jooli, Inc. You are expected to help manage the infrastructure at Jooli. Common tasks include provisioning resources for projects.

You are expected to have the skills and knowledge for these tasks, so step-by-step guides are not provided.

Some Jooli, Inc. standards you should follow:

1. Create all resources in the default region or zone, unless otherwise directed. The default region is `us-west4`, and the default zone is `us-west4-c`.
2. Naming normally uses the format *team-resource*; for example, an instance could be named **nucleus-webserver1**.
3. Make sure to create an instance template in `global` location.
4. Allocate cost-effective resource sizes. Projects are monitored, and excessive resource use will result in the containing project's termination (and possibly yours), so plan carefully. This is the guidance the monitoring team is willing to share: unless directed, use **e2-micro** for small Linux VMs, and use **e2-medium** for Windows or other applications, such as Kubernetes nodes.

# Your challenge

As soon as you sit down at your desk and open your new laptop, you receive several requests from the Nucleus team. Read through each description, and then create the resources.

</aside>

<aside>

# **Task 1. Create a project jumphost instance**

You will use this instance to perform maintenance for the project.

**Requirements:**

- Name the instance **`Instance name`**.
- Create the instance in the `ZONE` zone.
- Use an *e2-micro* machine type.
- Use the default image type (Debian Linux).

## My solution

### 1.- Declaring all the variables

```jsx
export PROJECT_ID=$(gcloud config get-value project)
export REGION=europe-west1
export ZONE=europe-west1-c
export INSTANCE_NAME=nucleus-jumphost-870
export TEMPLATE_NAME=nucleus-backend-template
export MIG_NAME=nucleus-web-mig
export FIREWALL_RULE_NAME=grant-tcp-rule-737
export HEALTH_CHECK_NAME=nucleus-health-check
export BACKEND_SERVICE_NAME=nucleus-backend-service
export URL_MAP_NAME=nucleus-url-map
export HTTP_PROXY_NAME=nucleus-http-proxy
export FORWARDING_RULE_NAME=nucleus-forwarding-rule
```

### 2.- Checking if region and zone are ok

```jsx
echo $REGION
echo $ZONE
```

### 3.- Configuring global region and zone

```jsx
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE
```

### 4.- Creating the instance

Name the instance **`Instance name,`** Create the instance in the `ZONE` zone. Use an *e2-micro* machine type. Use the default image type (Debian Linux)

```jsx
gcloud compute instances create $INSTANCE_NAME \
    --zone=$ZONE \
    --machine-type=e2-micro \
    --image-family=debian-11 \
    --image-project=debian-cloud
```

### 5.- I refresh the page to check if the instance was created

```jsx
gcloud compute instances list --zone=$ZONE
```

```jsx
gcloud compute instances describe $INSTANCE_NAME --zone=$ZONE
```

</aside>

<aside>

# **Task 2. Set up an HTTP load balancer**

You will serve the site via nginx web servers, but you want to ensure that the environment is fault-tolerant. Create an HTTP load balancer with a managed instance group of **2 nginx web servers**. Use the following code to configure the web servers; the team will replace this with their own configuration later.

**Note:** There is a limit to the resources you are allowed to create in your project, so do not create more than 2 instances in your managed instance group. If you do, the lab might end and you might be banned.

```jsx
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF
```

You need to:

- Create an instance template. Don't use the default machine type. Make sure you specify **e2-medium** as the machine type and create the **Global** template.
- Create a managed instance group based on the template.
- Create a firewall rule named as ____ to allow traffic (80/tcp).
- Create a health check.
- Create a backend service and add your instance group as the backend to the backend service group with named port (http:80).
- Create a URL map, and target the HTTP proxy to route the incoming requests to the default backend service.
- Create a target HTTP proxy to route requests to your URL map
- Create a forwarding rule.

## My solution

### 1.-  Setting the script

```jsx
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF
```

### 2.-  Creating the instance template -Here the problems started, it doesn’t accept the flag —global

Create an instance template. Don't use the default machine type. Make sure you specify **e2-medium** as the machine type and create the **Global** template. Make sure to create an instance template in `global` location.

*I deleted the “global” flag, the reason is by default it is global, no need to specify 

```jsx
gcloud compute instance-templates create $TEMPLATE_NAME \
   --machine-type=e2-medium \
   --image-family=debian-11 \
   --image-project=debian-cloud \
   --metadata-from-file=startup-script=startup.sh \
   --tags=allow-health-check
```

Checking the instance template it is ok

```jsx
gcloud compute instance-templates list
```

```jsx
gcloud compute instance-templates describe $TEMPLATE_NAME
```

### 3.- Trying to update the GC version: Google Cloud SDK 513.0.0

```jsx
gcloud version
```

### 4.- Creating the managed instance group

Create a managed instance group based on the template. MIG

```jsx
gcloud compute instance-groups managed create $MIG_NAME \
  --template=$TEMPLATE_NAME \
  --size=2 \
  --zone=$ZONE
```

Set the named ports for the managed instance group

```jsx
gcloud compute instance-groups managed set-named-ports $MIG_NAME \
  --named-ports=http:80 \
  --zone=$ZONE
```

### 5.- Creating the firewall rule

Create a firewall rule named as `Firewall rule` to allow traffic (80/tcp)

```jsx
gcloud compute firewall-rules create $FIREWALL_RULE_NAME \
  --allow=tcp:80 \
  --target-tags=allow-health-check
```

### 6.- The health check

```jsx
gcloud compute health-checks create http $HEALTH_CHECK_NAME \
  --port=80
```

### 7.- The backend service

Create a backend service and add your instance group as the backend to the backend service group with named port (http:80).

```jsx
gcloud compute backend-services create $BACKEND_SERVICE_NAME \
  --protocol=HTTP \
  --health-checks=$HEALTH_CHECK_NAME \
  --global
```

### 8.- The instance group as the backend

```jsx
  gcloud compute backend-services add-backend $BACKEND_SERVICE_NAME \
	  --instance-group=$MIG_NAME \
	  --instance-group-zone=$ZONE \
	  --balancing-mode=UTILIZATION \
	  --max-utilization=0.8 \
	  --global
```

### 9.- The url map

Create a URL map, and target the HTTP proxy to route the incoming requests to the default backend service.

```jsx
gcloud compute url-maps create $URL_MAP_NAME \
  --default-service=$BACKEND_SERVICE_NAME
```

### 10.- The HTTP proxy

Create a target HTTP proxy to route requests to your URL map

```jsx
gcloud compute target-http-proxies create $HTTP_PROXY_NAME \
  --url-map=$URL_MAP_NAME
```

### 11.- The forwarding rule

```jsx
gcloud compute forwarding-rules create $FORWARDING_RULE_NAME \
  --target-http-proxy=$HTTP_PROXY_NAME \
  --ports=80 \
  --global
```

### 12.- Checking if everything goes well

```jsx
gcloud compute instances list
gcloud compute instance-templates list
gcloud compute instance-groups managed list --zones=$ZONE
gcloud compute firewall-rules list
gcloud compute health-checks list --global
gcloud compute backend-services list --global
gcloud compute url-maps list --global
gcloud compute target-http-proxies list --global
gcloud compute forwarding-rules list --global
```

```jsx
curl http://[EXTERNAL IP 34.]
```

```jsx
curl http://[EXTERNAL LOAD BALANCER 35.]
```

</aside>
