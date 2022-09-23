https://blog.searce.com/disaster-recovery-of-gke-cluster-using-backup-for-gke-32a8160b5d66#:~:text=Disaster%20Recovery%20of%20GKE%20cluster%20using%20Backup%20for,...%203%20Let%E2%80%99s%20Get%20Started%20%21%21%20Prerequisites%3A%20


Disaster Recovery of GKE cluster using Backup for GKE. For complete Demo understanding, refer above link for detailed understanding

Disaster Recovery of GKE cluster using Backup for GKE
No Worries for GKE Workloads :)
In this blog I’ll demonstrate how we can use Backup for GKE to create a DR of GKE cluster containing EFK stack as application workload.

GKE (Google Kubernetes Engine) service is widely used by organizations for deploying, managing, and scaling your containerized applications using Google infrastructure. With this wide usage comes a need to make the GKE cluster fault-tolerant.

Introduction
Google has recently launched the Preview for Backup for GKE, a simple, cloud-native way to secure, manage, and restore your microservices workload and storage.

Backups of your GKE workloads can be scheduled periodically for both application data (Volume backups) and GKE cluster state data (Config backup) which can be useful for disaster recovery, CI/CD pipelines, cloning workloads, or upgrade scenarios.

You can also restore each backup to a cluster within the same region or, to a cluster in a different region. It is majorly used for restoring stateful workload on GKE in a simpler and faster way.

With Backup for GKE, you’ll be able to simply meet your service-level objectives, automate common backup and recovery tasks, and show reporting for compliance and audit purposes.


![image](https://user-images.githubusercontent.com/72337263/191938766-d7ad2b17-ad81-4411-859d-4bc8fa0da7c3.png)


The two main components of Backup for GKE architecture are:

Service
Agent
What is service ?
The Backup for GKE service provides a consumer API endpoint for users to interact with it using gcloud-sdk or console.
It’s a control plane for Backup for GKE.
Service API methods are used for CRUD operations against the resources managed by Backup for GKE.
Resources managed by Backup for GKE are:
Backup : Represents the backup for a particular portion or entire GKE cluster at a specific point in time.
Restore : Represents the restore of a specific portion of a particular backup into a GKE cluster.
BackupPlan : The parent resource for Backup resources that represent a chain of backups.
RestorePlan : It provides a reusable restore template.
What is Agent ??
An agent runs as a pod in every GKE cluster that you configure to be backed up by the Backup for GKE service.
It performs backup and restore operations by interacting with Agent API provided by Backup for GKE service.
Let’s Get Started !!
Prerequisites:
Enable the Google Kubernetes Engine API.
Install the Google Cloud CLI.
Enable the Backup for GKE API.
Ensure that your user account or service account has the below permissions:
Kubernetes Engine Admin.
Backup for GKE Admin
GitHub
You can find the resources at abhivaidya07/efk_stack

Create a GKE cluster
To create a GKE cluster for backup you need to enable one add-on named as BackupRestore.
Using Backup for GKE feature requires the Workload Identity feature to be enabled first.
Use the following command to deploy a GKE cluster with Backup for GKE enabled.
gcloud beta container --project "<PROJECT_ID>" clusters create "gke-backup-cluster" --zone "<ZONE>" --machine-type "e2-standard-4" --num-nodes "2" --network "<VPC_NETWORK>" --subnetwork "<SUBNETWORK>" --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver,BackupRestore --workload-pool "<PROJECT_ID>.svc.id.goog"
Note: This cluster is deployed with the minimum required configuration.

The above command creates a GKE cluster with Backup for GKE feature enabled and a pod gkebackup-agent-0 starts running inside the cluster in gkebackup namespace, which acts an agent.
Replace the following:

PROJECT_ID : The ID of your Google Cloud project.
ZONE : The zone in which you want to deploy GKE cluster.
VPC_NETWORK : The name of VPC network in which you want to deploy GKE cluster.
SUBNETWORK : The name of sub-network in which you want to deploy GKE cluster.
Install the kubectl command.
gcloud components install kubectl
To generate a kubeconfig context for the created cluster, run the following command:
gcloud container clusters get-credentials gke-backup-cluster --zone <ZONE> --project <PROJECT_ID>
Replace the following:

PROJECT_ID : The ID of your Google Cloud project.
ZONE : The zone in which GKE cluster is deployed.
Deploy EFK Stack
EFK stack is a set of tools for log aggregation and analysis.
Elasticsearch
Elasticsearch is a distributed object storage and scalable search engine. It excels in indexing semi-structured data such as logs. It stores data as JSON documents and each document is a set of key-value pairs.
Fluentd
Fluentd is used to collect, transform and ship log data to Elasticsearch backend. It tails the container log files and transform the log data and deliver it to Elasticsearch cluster, where it will be indexed and stored.
Kibana
Kibana is a data visualization frontend and dashboard for Elasticsearch.
We will deploy EFK stack in the GKE cluster and load some data in it.
Create a kube-logging namespace for EFK stack by copying the below contents into a file named as namespace.yaml

Apply the manifest file.
kubectl apply -f namespace.yaml
In this blog, we will use 3 Elasticsearch Pods to avoid the “split-brain” issue that occurs in highly-available multi-node clusters.
First we will create a headless-service which defines DNS domain for the 3 pods. Copy the below contents into a file named as elasticsearch_service.yaml

Apply the manifest file.
kubectl apply -f elasticsearch_service.yaml
We will use Kubernetes StatefulSet to deploy Elasticsearch cluster because elasticsearch requires stable identity to pods and stable storage. Copy the below contents into a file named as elasticsearch_statefulset.yaml

Apply the manifest file.
kubectl apply -f elasticsearch_statefulset.yaml
Monitor the StatefulSet as it is rolled out, using the following command:
kubectl rollout status sts/es-cluster -n kube-logging
Wait until all 3 pods are ready and the rollout is completed.
Check the status/health of the Elasticsearch cluster by forwarding its port to the local port.
kubectl port-forward es-cluster-0 9200:9200 -n kube-logging
Visit the following URL on web browser:
http://localhost:9200/_cluster/health?pretty
You will see the similar output
{
  "cluster_name" : "k8s-logs",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
The above output shows that the cluster is up and running with status as green.
Deploy Kibana for data visualisation. Create a file named as kibana.yaml and copy the below contents into it.

Apply the manifest file.
kubectl apply -f kibana.yaml
Check the rollout status using following command:
kubectl rollout status deployment/kibana -n kube-logging
Wait until the kibana deployment is rolled out successfully.
Verify Kibana application is running.
kubectl get pods -n kube-logging
Access the Kibana UI by forwarding its port to the local port.
Run the below command on another terminal:
kubectl port-forward <POD_NAME> 5601:5601 -n kube-logging
Replace the following:

POD_NAME : Name of the Kibana pod.
Visit the following URL on web browser:
http://localhost:5601
You will see the Kibana welcome page.

Now we will deploy Fluentd DaemonSet, which will roll out Fluentd logging agent pod on every node in our GKE cluster. Fluentd DaemonSet uses a node-level logging approach to collect the logs from GKE nodes.
Create a file named as fluentd.yaml and copy the below contents in it:

Apply the manifest file.
kubectl apply -f fluentd.yaml
Verify the DaemonSet is rolled out successfully.
kubectl get ds -n kube-logging
You should see the following status output:

Now we will use Kibana to verify whether Fluentd is properly collecting and shipping the log data to Elasticsearch.
If port-forward command for Kibana pod is still running visit http://localhost:5601 , if not run the following command on another terminal again:
kubectl port-forward <POD_NAME> 5601:5601 -n kube-logging
Replace the following:

POD_NAME : Name of the Kibana pod.
Visit the following URL on web browser:
http://localhost:5601
Click on Discover in the left-hand navigation menu.

We’ll use the logstash-* index pattern to capture all the log data in our Elasticsearch cluster. Enter logstash-* in the text box and click on Next step.

In Step 2 of 2 select timestamp which allows log data to be filtered by time and click on Create index pattern.

Now, again hit Discover in the left hand navigation menu.

You will see a histogram graph and log entries for last 15 minutes.
Now we’ll generate some load on our GKE cluster.
We will deploy a simple httpd web-server. Create a file named as httpd.yaml and copy the below contents in it:

Apply the manifest file.
kubectl apply -f httpd.yaml
Wait till the External-IP is assigned to the service.
kubectl get svc -n httpd
Let’s hit the web-server with some simultaneous requests. Run the below command on the terminal:
for i in {1..40}; do curl <EXTERNAL_IP>; done
Replace the following:

EXTERNAL_IP : External IP of the webserver service.
Go back to the Discover page in Kibana Dashboard, in the search bar type kubernetes.container_name : apache and click Update. This filters the log data for container named as apache.

As you can see in the above output data has been generated for our httpd web-server. You can get additional metadata like the pod name, Kubernetes node, Namespace, and more by clicking on the log entries.
Please note the timestamp for the log entries. In the above output logs are generated for httpd web-server GET request before Jun 12, 2022 @ 14:50:47.854.
Create Backup Plan
A backup plan for GKE cluster contains a backup configuration including the source cluster, the workloads and the container images to back up from the source cluster.
One GKE cluster can have multiple backup plan. It is recommended to have at least one backup plan for each GKE cluster.
Backup for GKE supports limited number of regions. You can find out the regions available for backup plans, backups, and restores by using the below Google Cloud CLI command:
gcloud alpha container backup-restore locations list \
    --project=<PROJECT_ID>
Replace the following:

PROJECT_ID : The ID of your Google Cloud project.
Now let’s create the backup for our EFK stack GKE cluster.
Go to the Google Kubernetes Engine page in the Google Cloud console.
From the left navigation menu, click on Backup for GKE.
On the top, click Create a backup plan.
In the Plan details section, select the appropriate cluster from the dropdown list, which in our case is gke-backup-cluster.
Enter the Backup plan name for e.g. gke-backup-cluster-bkp-plan.
(Optional) Add some description.
Select the Region for the backup for e.g. us-west1.
And click NEXT.

(Optional) In the Scope and encryption section, select which part of the cluster you want to backup.
Select Entire cluster if you want to backup all namespaces.
If you want only selected namespaces to backup, select Selected namespaces within this cluster and select the namespaces from the dropdown list.
If you want specific applications in the backup, select Selected protected applications within this cluster to add resources by specifying the namespace and application name.
In our case, select Entire cluster option.
Select the Include Secrets checkbox, if you want to include secrets. In our case not required.
Select the Persistent Volume Data checkbox, as we want our application data to be backed up.
Keep Encryption as default i.e. Google-managed encryption key. To add another layer of encryption use customer-managed key.
And click NEXT.

(Optional) In the Schedule and retention section, keep everything as default.
Backup for GKE provides us an option to create Periodic backups, where CRON job can be scheduled for the backup plan using standard cron syntax. For example, 12 05 * * * creates a backup at 12:05 PM every day.
But for now we are not creating any CRON job for this backup plan.
Click NEXT.

Review the backup plan details and click Create plan.

Backup Workloads
Now we are creating manual backup for the backup plan using Google Cloud console.
Go to the Google Kubernetes Engine page in the Google Cloud console.
From the left navigation menu, click on Backup for GKE.
Click the Backup Plans tab.
Select the backup plan created in previous stage.

Click Start a backup.
Enter the Backup name for e.g. gke-backup-cluster-bkp
(Optional) Add some description.
(Optional) Set the number of days after which the backups will be automatically deleted.
(Optional) Set the number of days during which the backups cannot be deleted.
Click Start backup.

To view the created backup, Click the Backups tab.
You will see a backup is getting generated, with status In progress.
Wait until it the status becomes Succeeded.

Create a Target Cluster
We will now create a target GKE cluster where our backup cluster can restore.
The target GKE cluster should be in the region supported by Backup for GKE and also have Backup for GKE feature enabled.
Use the following command to create a target GKE cluster:
gcloud beta container --project "<PROJECT_ID>" clusters create "gke-restore-cluster" --zone "<ZONE>" --machine-type "e2-standard-4" --num-nodes "2" --network "<VPC_NETWORK>" --subnetwork "<SUBNETWORK>" --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver,BackupRestore --workload-pool "<PROJECT_ID>.svc.id.goog"
Replace the following:

PROJECT_ID : The ID of your Google Cloud project.
ZONE : The zone (Supported by Backup for GKE) in which you want to deploy the target GKE cluster.
VPC_NETWORK : The name of VPC network in which you want to deploy target GKE cluster.
SUBNETWORK : The name of sub-network in which you want to deploy target GKE cluster.
Create Restore Plan
Restore plans can be used for restoring your backups of GKE cluster as they are the pre-configured scenarios for a corresponding line of backups. We can quickly restore a backup when any incident occurs.
To create a Restore Plan, go to the Google Kubernetes Engine page in the Google Cloud console.
From the left navigation menu, click on Backup for GKE.
Click Create a restore plan.
In the Name your plan and choose a cluster section, give Name to the restore plan for e.g. gke-backup-cluster-restore-plan
(Optional) Add some description.
Select the Backup plan from the dropdown that we created in earlier stage. i.e. gke-backup-cluster-bkp-plan
Select a target cluster from the dropdown created in previous stage. i.e. gke-restore-cluster
Click NEXT

In the Choose namespaced resources section, keep everything as default and click NEXT.

In the Choose cluster-scoped resources section, Click Add Groupkind and add the following API groups:
API group : rbac.authorization.k8s.io
Object kind : ClusterRole
API group : rbac.authorization.k8s.io
Object kind : ClusterRoleBinding
The reason why we are adding this is because the cluster-scoped resources like ClusterRole and ClusterRoleBinding are not restored by default.
Fluentd application creates a ClusterRole and ClusterRoleBinding Kubernetes resources which are required to be restored, so that our Fluentd application comes in running state.
Select the Overwrite resources in target cluster checkbox to delete a resource if it already exists in the target cluster, and restore the copy from the backup.

(Optional) The Add substitution rules section can be skipped as we do not need any substitution rules.
Click Create plan.
You will see the details of the created Restore Plan.

Restore GKE Workloads
Let’s assume that some incident occurred and your GKE cluster having all application workloads gets down, which in our case is gke-backup-cluster.
In such cases, we need to restore our GKE cluster using the restore plan created in the previous stage.
Go to the Google Kubernetes Engine page in the Google Cloud console.
From the left navigation menu, click on Backup for GKE.
Click the Backups tab.
Now find the backup that we created in the earlier stage. i.e. gke-backup-cluster-bkp

Click on Set up a restore.
Choose a restore plan from the dropdown list that we created in previous stage. i.e. gke-backup-cluster-restore-plan
Give a Name to the restore for e.g. gke-backup-cluster-restore
(Optional) Add some description.
Click Restore.

Wait until the backup is restored.

When the backup is restored, the application workloads and volumes from the previous cluster i.e. gke-backup-cluster are re-created in the target cluster i.e. gke-restore-cluster.
Validate Target Cluster
To validate the target cluster, we need to first connect to the cluster using following command:
gcloud container clusters get-credentials gke-restore-cluster --zone <ZONE> --project <PROJECT_ID>
Replace the following:

PROJECT_ID : The ID of your Google Cloud project.
ZONE : The zone in which the target GKE cluster is deployed.
After connecting to the target cluster, check if all the monitoring applications are in running state.
kubectl get pods -n kube-logging
You should see similar output:

Also check for the httpd web-server application.
kubectl get pods -n httpd
You should see similar output:

Once all the applications are in running state, validate if the logs from the previous cluster i.e. gke-backup-cluster are also restored.
To do so, access the Kibana UI by forwarding its port to the local port.
Run the below command on another terminal:
kubectl port-forward <POD_NAME> 5601:5601 -n kube-logging
Replace the following:

POD_NAME : Name of the Kibana pod.
Visit the following URL on web browser:
http://localhost:5601
You will see the Kibana welcome page.
Click on Discover in the left-hand navigation menu.
You will see the logs for target cluster i.e. gke-restore-cluster, are getting created automatically, without creating any index pattern. This means our index pattern is restored from the backup cluster i.e. gke-backup-cluster.

Let’s check if logs of our httpd web-server that we generated in backup cluster i.e. gke-backup-cluster for testing are restored or not.
The timestamp we noted for httpd web-server GET request was Jun 12, 2022 @ 14:50:47.854.
In the search bar type kubernetes.container_name : apache
Now click Show dates, and select the start and end time for log entries.

In our case logs for httpd web-server GET request were generated before Jun 12, 2022 @ 14:50:47.854, so i selected a time frame between June 12, 2022 14:00 to 15:00.
Click Update.

Logs for our httpd web-server application are also restored successfully.
After clicking on the log-entries, you will get metadata like pod-name, Kubernetes node and many more which confirms that the logs are restored from the previous cluster i.e. gke-backup-cluster.
Conclusion
In this blog we’ve covered how Backup for GKE can be used to perform disaster recovery by restoring the application workloads and volumes from the previous GKE cluster to a new target GKE cluster in the different region with few clicks when an incident occurs.

Backup for GKE provides data loss prevention for GKE clusters which is useful while CI/CD pipelines, cloning workloads, or upgrade scenarios.

References
Backup for GKE
