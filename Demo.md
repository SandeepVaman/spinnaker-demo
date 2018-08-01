# Continuous Delivery Pipelines with Spinnaker and Kubernetes Engine
This tutorial shows you how to create a continuous delivery pipeline using Kubernetes Engine, Cloud Source Repositories, Container Builder, Resource Manager, and Spinnaker. After you create a sample application, you configure these services to automatically build, test, and deploy it. When you modify the application code, the changes trigger the continuous delivery pipeline to automatically rebuild, retest, and redeploy the new version.

## Pipeline architecture

<img src="images/spinnacker project architecture.png" alt="pipeline architecture">

To continuously deliver application updates to your users, you need an automated process that reliably builds, tests, and updates your software. Code changes should automatically flow through a pipeline that includes artifact creation, unit testing, functional testing, and production rollout. In some cases, you want a code update to apply to only a subset of your users, so that it is exercised realistically before you push it to your entire user base. If one of these canary releases proves unsatisfactory, your automated procedure must be able to quickly roll back the software changes.

With Kubernetes Engine and Spinnaker, you can create a robust continuous delivery flow that helps to ensure your software is shipped as quickly as it is developed and validated. Although rapid iteration is your end goal, you must first ensure that each application revision passes through a series of automated validations before becoming a candidate for production rollout. When a given change has been vetted through automation, you can also validate the application manually and conduct further prerelease testing.

After your team decides the application is ready for production, one of your team members can approve it for production deployment.

## Application delivery pipeline
In this tutorial, you build the continuous delivery pipeline shown in the following diagram.

<img src="images/what we will do.svg" alt="What we will do"/>

## Objectives
* Set up your environment by launching Cloud Shell, creating a Kubernetes Engine cluster, and configuring your identity and user management scheme.
* Download a sample application, create a Git repository, and upload it to a Cloud Source Repository.
* Deploy Spinnaker to Kubernetes Engine using Helm.
* Build your Docker image.
* Create triggers to create Docker images when your application changes.
* Configure a Spinnaker pipeline to reliably and continuously deploy your application to Kubernetes Engine.
* Deploy a code change, triggering the pipeline, and watch it roll out to production.

## Cost
This tutorial uses billable components of Google Cloud Platform (GCP), including:
* Kubernetes Engine
* Cloud Load Balancer
* Container Builder
* Resource Manager

Use the Pricing Calculator to generate a cost estimate based on your projected usage.


Before you begin
1. Select or create a GCP project.
2. Make sure that billing is enabled for your project.
3. Enable the Kubernetes Engine, Container Builder, and Resource Manager APIs.

## Set up your environment
In this section, you configure the infrastructure and identities required to complete the tutorial.

Start a Cloud Shell instance and create a Kubernetes Engine cluster
You run all the terminal commands in this tutorial from Cloud Shell.

1. Open Cloud Shell:

2. Create a Kubernetes Engine cluster to deploy Spinnaker and the sample application with the following commands:
```
gcloud config set compute/zone us-central1-f
```
```
gcloud container clusters create spinnaker-tutorial \
    --machine-type=n1-standard-2
 ```
## Configure identity and access management
You create a Cloud Identity and Access Management (Cloud IAM) service account to delegate permissions to Spinnaker, allowing it to store data in Cloud Storage. Spinnaker stores its pipeline data in Cloud Storage to ensure reliability and resiliency. If your Spinnaker deployment unexpectedly fails, you can create an identical deployment in minutes with access to the same pipeline data as the original.

1.Create the service account:

```
gcloud iam service-accounts create  spinnaker-storage-account \
    --display-name spinnaker-storage-account
```    
2. Store the service account email address and your current project ID in environment variables for use in later commands:
```
export SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:spinnaker-storage-account" \
    --format='value(email)')
export PROJECT=$(gcloud info --format='value(config.project)')
```
3. Bind the storage.admin role to your service account:
```
gcloud projects add-iam-policy-binding \
    $PROJECT --role roles/storage.admin --member serviceAccount:$SA_EMAIL
   ```
4. Download the service account key. You need this key later when you install Spinnaker and upload the key to Kubernetes Engine.
```
gcloud iam service-accounts keys create spinnaker-sa.json --iam-account $SA_EMAIL
```

## Deploying Spinnaker using Helm
n this section, you use Helm to deploy Spinnaker from the Charts repository. Helm is a package manager you can use to configure and deploy Kubernetes applications.

### Install Helm.
1. Download and install the helm binary:
```
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.7.2-linux-amd64.tar.gz
```

2. Unzip the file to your local system:
```
tar zxfv helm-v2.7.2-linux-amd64.tar.gz
cp linux-amd64/helm .
```

3. Grant Tiller, the server side of Helm, the cluster-admin role in your cluster:
```
kubectl create clusterrolebinding user-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
kubectl create serviceaccount tiller --namespace kube-system
kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```
4. Grant Spinnaker the cluster-admin role so it can deploy resources across all namespaces:
```
kubectl create clusterrolebinding --clusterrole=cluster-admin --serviceaccount=default:default spinnaker-admin
```
5. Initialize Helm to install Tiller in your cluster:
```
./helm init --service-account=tiller
./helm update
```

6. Ensure that Helm is properly installed by running the following command. If Helm is correctly installed, v2.7.2 appears for both client and server.
```
./helm version
```
```
Client: &version.Version{SemVer:"v2.7.2",
GitCommit:"8478fb4fc723885b155c924d1c8c410b7a9444e6",
GitTreeState:"clean"}Server: &version.Version{SemVer:"v2.7.2",
GitCommit:"8478fb4fc723885b155c924d1c8c410b7a9444e6",
GitTreeState:"clean"}
```
### Configure Spinnaker
1. Create a bucket for Spinnaker to store its pipeline configuration:
```
export PROJECT=$(gcloud info \
    --format='value(config.project)')
export BUCKET=$PROJECT-spinnaker-config
gsutil mb -c regional -l us-central1 gs://$BUCKET
```

2. Create the configuration file:
```
export SA_JSON=$(cat spinnaker-sa.json)
export PROJECT=$(gcloud info --format='value(config.project)')
export BUCKET=$PROJECT-spinnaker-config
cat > spinnaker-config.yaml <<EOF
storageBucket: $BUCKET
gcs:
  enabled: true
  project: $PROJECT
  jsonKey: '$SA_JSON'

# Disable minio as the default
minio:
  enabled: false


# Configure your Docker registries here
accounts:
- name: gcr
  address: https://gcr.io
  username: _json_key
  password: '$SA_JSON'
  email: 1234@5678.com
EOF
```

### Deploy the Spinnaker chart
1. Use the Helm command-line interface to deploy the chart with your configuration set. This command typically takes five to ten minutes to complete.
```
./helm install -n cd stable/spinnaker -f spinnaker-config.yaml --timeout 600 \
    --version 0.3.1
    ```
2. After the command completes, run the following command to set up port forwarding to the Spinnaker UI from Cloud Shell:
```
export DECK_POD=$(kubectl get pods --namespace default -l "component=deck" \
    -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward --namespace default $DECK_POD 8080:9000 >> /dev/null &
```
3. To open the Spinnaker user interface, click Web Preview in Cloud Shell and click Preview on port 8080.

4. You should see the welcome screen, followed by the Spinnaker UI:

<img src="" alt=""/>

## Building the Docker image
In this section, you configure Container Builder to detect changes to your application source code, build a Docker image, and then push it to Container Registry.

### Create your source code repository
1. Download the source code:
```
wget https://gke-spinnaker.storage.googleapis.com/sample-app.tgz
```
2. Unpack the source code:
```
tar xzfv sample-app.tgz
```
3. Change directories to source code:
```
cd sample-app
```

4. Set the username and email address for your Git commits in this repository. Replace [EMAIL_ADDRESS] with your Git email address, and replace [USERNAME] with your Git username.
```
git config --global user.email "[EMAIL_ADDRESS]"
git config --global user.name "[USERNAME]"
```

5. Make the initial commit to your source code repository:
```
git init
git add .
git commit -m "Initial commit"
```
6. Create a repository to host your code:
``
gcloud source repos create sample-app
git config credential.helper gcloud.sh
```
7. Add your newly created repository as remote:
```
export PROJECT=$(gcloud info --format='value(config.project)')
git remote add origin https://source.developers.google.com/p/$PROJECT/r/sample-app
```
8. Push your code to the new repository's master branch:
```
git push origin master
```
9. Check that you can see your source code in the console:

Configure your build triggers
In this section, you configure Container Builder to build and push your Docker images every time you push Git tags to your source repository. Container Builder automatically checks out your source code, builds the Docker image from the Dockerfile in your repository, and pushes that image to Container Registry.
<img src="cloud-build" alt="cloud build">
1. In the GCP Console, click Build Triggers in the Container Registry section.

2. Select Cloud Source Repository and click Continue.

3. Select your newly created sample-app repository from the list, and click Continue.
4. Set the following trigger settings:

* Name:sample-app-tags
* Trigger type: Tag
* Tag (regex): v.*
* Build configuration: cloudbuild.yaml
* cloudbuild.yaml location: /cloudbuild.yaml
5. Click Create trigger.

From now on, whenever you push a Git tag prefixed with the letter "v" to your source code repository, Container Builder automatically builds and pushes your application as a Docker image to Container Registry.

### Build your image
Push your first image using the following steps:

1. Go to your source code folder in Cloud Shell.
2. Create a Git tag:
```
git tag v1.0.0
```
3. Push the tag:
```
git push --tags
```
4. In Container Registry, click Build History to check that the build has been triggered. If not, verify the trigger was configured properly in the previous section.

## Configuring your deployment pipelines
Now that your images are building automatically, you need to deploy them to the Kubernetes cluster.

You deploy to a scaled-down environment for integration testing. After the integration tests pass, you must manually approve the changes to deploy the code to production services.

### Create the application
1. In the Spinnaker UI, click Actions, then click Create Application.

<img src="Create an application" alt="Create an application">

2. In the New Application dialog, enter the following fields:

* Name: sample
* Owner Email: [your email address]
3. Click Create.

### Create service load balancers
To avoid having to enter the info

In Cloud Shell, run the following command from the sample-app root directory:
```
kubectl apply -f k8s/services
```
### Create the deployment pipeline
Next, you create the continuous delivery pipeline. In this tutorial, the pipeline is configured to detect when a Docker image with a tag prefixed with "v" has arrived in your Container Registry.

1. In a new tab of Cloud Shell, run the following command in the source code directory to upload an example pipeline to your Spinnaker instance:
```
export PROJECT=$(gcloud info --format='value(config.project)')
sed s/PROJECT/$PROJECT/g spinnaker/pipeline-deploy.json | curl -d@- -X \
    POST --header "Content-Type: application/json" --header \
    "Accept: /" http://localhost:8080/gate/pipelines
    ```
2. In the Spinnaker UI, click Pipelines on the top navigation bar
3. Click Configure in the Deploy pipeline.
```
### Run your pipeline manually
The configuration you just created contains a trigger to start the pipeline when you push a new Git tag containing the prefix "v". In this section of the tutorial, you test the pipeline by running it manually. In the next section, you test it by pushing a Git tag and watching the pipeline run automatically.

1. Return to the Pipelines page by clicking Pipelines.
2. Click Start Manual Execution.
3. Select the v1.0.0 tag from the Tag drop-down list, then click Run.
4. After the pipeline starts, click Details to see more information about the build's progress. This section shows the status of the deployment pipeline and its steps. Steps in blue are currently running, green ones have completed successfully, and red ones have failed. Click a stage to see details about it.

After 3 to 5 minutes the integration test phase completes and the pipeline requires manual approval to continue the deployment.

5. Hover over the yellow "person" icon and click Continue.
Your rollout continues to the production frontend and backend deployments. It completes after a few minutes.

6. To view the app, click Load Balancers in the top right of the Spinnaker UI.
7. Scroll down the list of load balancers and click Default, under sample-frontend-prod
8. Scroll down the details pane on the right and copy your application's IP address by clicking the clipboard button on the Ingress IP.
9. Paste the address into your browser to view the production version of the application.

## Triggering your pipeline from code changes
In this section, you test the pipeline end to end by making a code change, pushing a Git tag, and watching the pipeline run in response. By pushing a Git tag that starts with "v", you trigger Container Builder to build a new Docker image and push it to Container Registry. Spinnaker detects that the new image tag begins with "v" and triggers a pipeline to deploy the image to canaries, run tests, and roll out the same image to all pods in the deployment.

1. Change the color of the app from orange to blue:
```
sed -i 's/orange/blue/g' cmd/gke-info/common-service.go
```
2. Tag your change and push it to the source code repository:
```
git commit -a -m "Change color to blue"
git tag v1.0.1
git push --tags
```
3. See the new build appear in the Container Builder Build History.

4. Click Pipelines to watch the pipeline start to deploy the image.

5. Observe the canary deployments. When the deployment is paused, waiting to roll out to production, start refreshing the tab that contains your application. Nine of your backends are running the previous version of your application, while only one backend is running the canary. You should see the new, blue version of your application appear about every tenth time you refresh.

6. After testing completes, return to the Spinnaker tab and approve the deployment.

7. When the pipeline completes, your application looks like the following screenshot. Note that the color has changed to blue because of your code change, and that the Version field now reads v1.0.1.

8. Optionally, you can roll back this change by reverting your previous commit. Rolling back adds a new tag (v1.0.2), and pushes the tag back through the same pipeline you used to deploy v1.0.1:
```
git revert v1.0.1
git tag v1.0.2
git push --tags
```





