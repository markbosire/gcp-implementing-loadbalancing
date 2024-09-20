# GCP HTTP Load Balancer Setup Tutorial

## Task 1: Create an instance

In this task, we'll create an instance

### Command:

```bash
gcloud compute instances create $INSTANCE \
    --zone=$ZONE \
    --machine-type=e2-micro
```

### Explanation:

- `gcloud compute instances create`: This is the base command to create a new Compute Engine instance.
- `$INSTANCE`: This variable should contain the name of your instance . Replace $INSTANCE with the specified INSTANCE name in the instruction
- `--zone=$ZONE`: Specifies the zone where the instance will be created. The `$ZONE` variable should contain the specified zone .Replace $ZONE with the specified zone name in the instruction
- `--machine-type=e2-micro`: Sets the machine type to e2-micro

This command creates a new Compute Engine instance with the specified name, in the given zone, using the e2-micro machine type. By default, it will use the Debian Linux image.

## Task 2: Set up an HTTP load balancer

In this task, we'll create an HTTP load balancer with a managed instance group of 2 nginx web servers.

### Step 1: Create the startup script

```bash
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF
```

This creates a startup script that will be used to configure the web servers. It updates the system, installs nginx, starts the nginx service, and customizes the default nginx welcome page.

### Step 2: Create an instance template

```bash
gcloud compute instance-templates create web-server-template \
        --metadata-from-file startup-script=startup.sh \
        --machine-type e2-medium \
        --region $REGION
```
Replace $REGION with the specified region name in the instruction
This command creates an instance template:
- `--metadata-from-file startup-script=startup.sh`: Specifies the startup script we created earlier.
- `--machine-type e2-medium`: Sets the machine type to e2-medium as required.
- `--region $REGION`: Specifies the region for the template.

### Step 3: Create a managed instance group

```bash
gcloud compute instance-groups managed create web-server-group \
        --base-instance-name web-server \
        --size 2 \
        --template web-server-template \
        --region $REGION
```

This creates a managed instance group:
Replace $REGION with the specified region name in the instruction
- `--base-instance-name web-server`: Sets the base name for instances in the group.
- `--size 2`: Specifies that the group should maintain 2 instances.
- `--template web-server-template`: Uses the template we created in step 2.

### Step 4: Create a firewall rule

```bash
gcloud compute firewall-rules create $FIREWALL \
        --allow tcp:80 \
        --network default
```
Replace $FIREWALL with the specified firewall name in the instruction
This creates a firewall rule to allow HTTP traffic (port 80) to our instances. Without this rule, the incoming web requests would be blocked by GCP's default firewall settings

### Step 5: Create a health check

```bash
gcloud compute http-health-checks create http-basic-check
```

This creates a basic HTTP health check to monitor the health of our instances.

### Step 6: Set named ports for the instance group

```bash
gcloud compute instance-groups managed \
        set-named-ports web-server-group \
        --named-ports http:80 \
        --region $REGION
```

This sets a named port for our instance group, which will be used by the load balancer.
Replace $REGION with the specified region name in the instruction

### Step 7: Create a backend service

```bash
gcloud compute backend-services create web-server-backend \
        --protocol HTTP \
        --http-health-checks http-basic-check \
        --global
```

This creates a backend service and associates it with our health check.

### Step 8: Add the instance group to the backend service

```bash
gcloud compute backend-services add-backend web-server-backend \
        --instance-group web-server-group \
        --instance-group-region $REGION \
        --global
```
Replace $REGION with the specified region name in the instruction
This adds our instance group as a backend to the backend service.

### Step 9: Create a URL map

```bash
gcloud compute url-maps create web-server-map \
        --default-service web-server-backend
```

This creates a URL map and sets our backend service as the default service.

### Step 10: Create a target HTTP proxy

```bash
gcloud compute target-http-proxies create http-lb-proxy \
        --url-map web-server-map
```

This creates a target HTTP proxy and associates it with our URL map.

### Step 11: Create a forwarding rule

```bash
gcloud compute forwarding-rules create http-content-rule \
      --global \
      --target-http-proxy http-lb-proxy \
      --ports 80
```

This creates a forwarding rule that directs incoming traffic to our HTTP proxy.

### Step 12: List forwarding rules

```bash
gcloud compute forwarding-rules list
```

This lists all forwarding rules, allowing you to verify that your rule has been created successfully.

## Conclusion



Remember to wait for 5 to 7 minutes after completing these steps for the changes to propagate and for the lab to recognize completion of the task.
