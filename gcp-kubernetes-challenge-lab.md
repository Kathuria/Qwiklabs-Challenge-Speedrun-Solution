# How to solve Kubernetes in Google Cloud:

## Challenge scenario

You have just completed training on containers and their creation and management
and now you need to demonstrate to the Jooli Inc. development team your new skills.
You have to help with some of their initial work on a new project around an
application environment utilizing Kubernetes. Some of the work was already done for
you, but other parts require your expert skills.
You are expected to create container images, store the images in a repository, and
configure a Jenkins CI/CD pipeline to automate the build for the product. Your know
that Kurt, your supervisor, will ask you to complete these tasks:
Create a Docker image and store the Dockerfile.
Test the created Docker image.
Push the Docker image into the Container Repository.
Use the image to create and expose a deployment in Kubernetes
Update the image and push a change to the deployment.
Create a pipeline in Jenkins to deploy a new version of your image when the
source code changes.

Some Jooli Inc. standards you should follow:
Create all resources in the us-east1 region and us-east1-b zone, unless otherwise
directed.
Use the project VPCs.
Naming is normally _team-resource_ , e.g. an instance could be named **kraken-
webserver**.
Allocate cost effective resource sizes. Projects are monitored and excessive
resource use will result in the containing project’s termination (and possibly
yours), so beware. This is the guidance the monitoring team is willing to share:
unless directed, use n1-standard-1.

# Solution :

# Task 1: Create a Docker image and store the

## Dockerfile

open your cloud shell and start typing :
Open Cloud Shell and run source <(gsutil cat gs://cloud-
training/gsp318/marking/setup_marking.sh). This command will install marking scripts
you can use to help check your progress.
Use Cloud Shell to clone the valkyrie-app source code repository (it is in your
project).
The app source code is in valkyrie-app/source. Create valkyrie-app/Dockerfile and
add the configuration below.
```
gsutil cat gs://cloud-training/gsp318/marking/setup_marking.sh | bash
gcloud source repos clone valkyrie-app
cd valkyrie-app
cat > Dockerfile <<EOF
FROM golang:1.10
WORKDIR /go/src/app
COPY source.
RUN go install -v
ENTRYPOINT [“app”,”-single=true”,”-port=8080"]
EOF
docker build -t valkyrie-app:v0.0.1 .
step1.sh
```

# Task 2: Test the created Docker image

Launch a container using the image **valkyrie-app:v0.0.1**. You need to map the
host’s port 8080 to port 8080 on the container. Add & to the end of the command to
cause the container to run in the background.
When your container is running you will see the page by **Web Preview**.
Once you have your container running, and before clicking **Check my progress** ,
run step2.sh to perform the local check of your work. After you get a successful
response from the local marking you can check your progress.

```
docker run -p 8080:8080 valkyrie-app:v0.0.1 &
step2.sh
```

# Task 3: Push the Docker image in the

## Container Repository

Push the Docker image **valkyrie-app:v0.0.1** into the Container Registry.
Make sure you re-tag the container to gcr.io/YOUR_PROJECT/valkyrie-
app:v0.0.1

```
docker tag valkyrie-app:v0.0.1 gcr.io/$DEVSHELL_PROJECT_ID/valkyrie-app:v0.0.1
docker push gcr.io/$DEVSHELL_PROJECT_ID/valkyrie-app:v0.0.1
```

# Task 4: Create and expose a deployment in

## Kubernetes

Kurt created the deployment.yaml and service.yaml to deploy your new container
image to a Kubernetes cluster (called valkyrie-dev). The two files are in valkyrie-
app/k8s.
Remember you need to get the Kubernetes credentials before you deploy the image
onto the Kubernetes cluster.
Before you create the deployments make sure you check
the deployment.yaml and service.yaml files. Kurt thinks they need some values set (he
thinks he left some placeholder values).
You can check the load balancer once it’s available.

```
sed -i s#IMAGE_HERE#gcr.io/\$DEVSHELL_PROJECT_ID/valkyrie-app:v0.0.1#gk8s/deployment.yaml
gcloud container clusters get-credentials valkyrie-dev --zone us-east1-d
kubectl create -f k8s/deployment.yaml
kubectl create -f k8s/service.yaml
```

# Task 5: Update the deployment with a new

Before deploying the new code, increase the replicas from 1 to 3 to ensure you don’t
cause an outage.
Kurt made changes to the source code (he put the changes in a branch called **kurt-
dev** ). You need to merge **kurt-dev** into **master** (you should use git merge
origin/kurt-dev).
Build the new code as version v0.0.2 of valkyrie-app, push the updated image to the
Container Repository, and then redeploy to the valkyrie-dev cluster. You will know
you have the new v0.0.2 version because the titles for the cards will be green.

```
git merge origin/kurt-dev
kubectl edit deployment valkyrie-dev
```
<em>change replicas from 1 to 3</em>
```
docker build -t gcr.io/$DEVSHELL_PROJECT_ID/valkyrie-app:v0.0.2 .
docker push gcr.io/$DEVSHELL_PROJECT_ID/valkyrie-app:v0.0.2
```
<em>vim commands</em>

**i - insert, esc - enter command mode, :wq - write and quit (save/exit)**

```
kubectl edit deployment valkyrie-dev
```
<em>change 0.0.1 to 0.0.2 in two places</em>

# Task 6: Create a pipeline in Jenkins to deploy

This process of building the container and pushing to the container repository can be
automated using Jenkins. There is a Jenkins deployment in your valkyrie-dev cluster

- connect to Jenkins and configure a job to build when you push a change to the
  source code.
  Remember with Jenkins:
  Get the password with printf \$(kubectl get secret cd-jenkins -o jsonpath="
  {.data.jenkins-admin-password}" | base64 --decode);echo.
  Connect to the Jenkins console using the commands below (but make sure you
  don’t have a running container docker ps; if you do, kill it):
  Make two changes to your files before you commit and build:
  Edit valkyrie-app/Jenkinsfile and change YOUR_PROJECT to your actual
  project id.
  Edit valkyrie-app/source/html.go and change the two occurrences of green to
  orange.
  Use git to:
  Add all the changes then commit those changes to the master branch.
  Push the changes back to the repository.
  When you are ready, manually trigger a build (the initial build will take some time, so
  just monitor the process). The build will replace the running containers with
  containers with different tags; you will see orange colored headings.

```
docker ps
//get container id
docker kill <container_id>
export POD_NAME=$(kubectl get pods --namespace default -l
“app.kubernetes.io/component=jenkins-master” -l
“app.kubernetes.io/instance=cd” -o
jsonpath=”{.items[0].metadata.name}”)

// or use kubectl get pods -o wide and assign to POD_NAME=<jenkins_pod_name>

kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
kubectl get secret cd-jenkins -o jsonpath=”{.data.jenkins-admin-password}” | base64 --decode
```

> Open web-preview and login as admin with password from last command

> click credentials -> Jenkins -> Global Credentials

> Click add credentials

> select Google Service Account from metadata

> Click ok

> Click jenkins (top left)

> Click new item

> enter name ```valkyrie-app```

> select pipeline

> click ok

> select pipeline script from SCM

> Set SCM to Git

> Add the source code repo url (find it using ```gcloud source repos list```)

> Set credentials to <qwiklabs-id>
  
> Click save

> Go back to cloud console and do next steps

> Update project in Jenkinsfile

```
vim Jenkinsfile
```
> Change card Green Color to Orange
```
cd source
vim html.go
```

> Push changes to git
```
cd ..
git config --global user.email “you@example.com”
git config --global user.name “student”
git add.
git commit -m “build pipeline init”
git push
```

#### Initial build takes a while, just navigate to cloud-build -> Build History to complete

#### Then go to jenkins check applications -> pipelines -> builds and proceed with push to production of any running build

## Enjoy successful lab completion...
