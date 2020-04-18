# Cloud Engineering: Challenge Lab Solution
## GSP321

### Challenge scenario
As a cloud engineer in Jooli Inc. and recently trained with Google Cloud and Kubernetes you have been asked to help a new team (Griffin) set up their environment. The team has asked for your help and has done some work, but needs you to complete the work.

You are expected to have the skills and knowledge for these tasks so don’t expect step-by-step guides.

You need to complete the following tasks:

- Create a development VPC with three subnets manually
- Create a production VPC with three subnets using a provided Deployment Manager configuration
- Create a bastion that is connected to both VPCs
- Create a development Cloud SQL Instance and connect and prepare the WordPress environment
- Create a Kubernetes cluster in the development VPC for WordPress
- Prepare the Kubernetes cluster for the WordPress environment
- Create a WordPress deployment using the supplied configuration
- Enable monitoring of the cluster via stackdriver
- Provide access for an additional engineer

Some Jooli Inc. standards you should follow:
- Create all resources in the `us-east1` region and `us-east1-b` zone, unless otherwise directed.
- Use the project VPCs.
- Naming is normally team-resource, e.g. an instance could be named **kraken-webserver1**.
- Allocate cost effective resource sizes. Projects are monitored and excessive resource use will result in the containing project's termination (and possibly yours), so beware. This is the guidance the monitoring team is willing to share; unless directed, use `n1-standard-1`.

  **Your region and zone maybe different**
```
gcloud config set compute/region us-east1
gcloud config set project [GCP Project ID]
```

### Task 1: Create development VPC manually
Create a VPC called `griffin-dev-vpc` with the following subnets only:

- `griffin-dev-wp`
    - IP address block: `192.168.16.0/20`
- `griffin-dev-mgmt`
    - IP address block: `192.168.32.0/20`
```
gcloud compute networks create griffin-dev-vpc --subnet-mode=custom
gcloud compute networks subnets create griffin-dev-wp --network=griffin-dev-vpc --region=us-east1 --range=192.168.16.0/20
gcloud compute networks subnets create griffin-dev-mgmt --network=griffin-dev-vpc --region=us-east1 --range=192.168.32.0/20
```


### Task 2: Create production VPC using Deployment Manager
Use Cloud Shell and copy all files from `gs://cloud-training/gsp321/dm`.

Check the Deployment Manager configuration and make any adjustments you need, then use the template to create the production VPC with the 2 subnets.
```
gsutil cp -r gs://cloud-training/gsp321/dm .
cd dm
sed -i 's/SET_REGION/us-east1/g' prod-network.yaml
gcloud deployment-manager deployments create advanced-configuration --config prod-network.yaml
```


### Task 3: Create bastion host
Create a bastion host with two network interfaces, one connected to `griffin-dev-mgmt` and the other connected to `griffin-prod-mgmt`. Make sure you can SSH to the host.
```
gcloud compute instances create bastion --zone=us-east1-b --machine-type=n1-standard-1 --network-interface subnet=griffin-dev-mgmt --network-interface subnet=griffin-prod-mgmt
gcloud compute firewall-rules create bastion-dev-ssh --direction=INGRESS --priority=1000 --network=griffin-dev-vpc --action=ALLOW --rules=icmp,tcp:22,tcp:3389 --source-ranges=0.0.0.0/0
gcloud compute firewall-rules create bastion-prod-ssh --direction=INGRESS --priority=1000 --network=griffin-prod-vpc --action=ALLOW --rules=icmp,tcp:22,tcp:3389 --source-ranges=0.0.0.0/0
```


### Task 4: Create and configure Cloud SQL Instance
Create a **MySQL Cloud SQL Instance** called `griffin-dev-db` in us-east1. Connect to the instance and run the following SQL commands to prepare the **WordPress** environment:
```
gcloud sql instances create griffin-dev-db --region=us-east1 --tier=db-n1-standard-1
gcloud sql users set-password root --host=% --instance griffin-dev-db --password password
gcloud sql connect griffin-dev-db --user=root --password password
```
After connect, run the following `SQL command`
```
CREATE DATABASE wordpress;
GRANT ALL PRIVILEGES ON wordpress.* TO "wp_user"@"%" IDENTIFIED BY "stormwind_rules";
FLUSH PRIVILEGES;
exit
```
These SQL statements create the worpdress database and create a user with access to the wordpress dataase.

You will use the *username* and *password* in **Task 6**.


### Task 5: Create Kubernetes cluster
Create a 2 node cluster (n1-standard-4) called `griffin-dev`, in the `griffin-dev-wp` subnet, and in zone `us-east1-b`.
```
gcloud container clusters create griffin-dev --num-nodes 2 --machine-type=n1-standard-4 --zone us-east1-b --network griffin-dev-vpc --subnetwork griffin-dev-wp
gcloud container clusters get-credentials griffin-dev --zone=us-east1-b
```


### Task 6: Prepare the Kubernetes cluster
Use Cloud Shell and copy all files from `gs://cloud-training/gsp321/wp-k8s`.

The **WordPress** server needs to access the MySQL database using the username and password you created in task 4. You do this by setting the values as secrets. **WordPress** also needs to store its working files outside the container, so you need to create a volume.

Add the following secrets and volume to the cluster using `wp-env.yaml`. Make sure you configure the username to `wp_user` and password to `stormwind_rules` before creating the configuration.

You also need to provide a key for a service account that was already set up. This service account provides access to the database for a sidecar container. Use the command below to create the key, and then add the key to the Kubernetes environment.
```
gsutil cp -r gs://cloud-training/gsp321/wp-k8s .
cd wp-k8s
sed -i 's/username_goes_here/wp_user/g' wp-env.yaml
sed -i 's/password_goes_here/stormwind_rules/g' wp-env.yaml
gcloud iam service-accounts keys create key.json --iam-account=cloud-sql-proxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
kubectl create secret generic cloudsql-instance-credentials --from-file key.json
kubectl create -f wp-env.yaml
```


### Task 7: Create a WordPress deployment
Now you have provisioned the MySQL database, and set up the secrets and volume, you can create the deployment using `wp-deployment.yaml`. Before you create the deployment you need to edit `wp-deployment.yaml` and replace **YOUR_SQL_INSTANCE** with griffin-dev-db's **Instance connection name**. Get the **Instance connection name** from your Cloud SQL instance.

After you create your WordPress deployment, create the service with `wp-service.yaml`.

Once the Load Balancer is created, you can visit the site and ensure you see the **WordPress** site installer. At this point the dev team will take over and complete the install and you move on to the next task.
```
sed -i 's/YOUR_SQL_INSTANCE/'$GOOGLE_CLOUD_PROJECT':us-east1:griffin-dev-db/g' wp-deployment.yaml
kubectl create -f wp-deployment.yaml
kubectl create -f wp-service.yaml
gcloud compute forwarding-rules list
```


### Task 8: Enable monitoring
Create an uptime check for your WordPress development site.

At Google Cloud Platform, navigate to **Monitoring**, and wait workspace to create.


### Task 9: Provide access for an additional engineer
You have an additional engineer starting and you want to ensure they have access to the project, so please go ahead and grant them the editor role to the project.

The second user account for the lab represents the additional engineer.
Add Project Editor role to 2nd user
```
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member user:[2nd_email] --role roles/editor
```

### Check all progress, if lucky, you got the **Badge**!