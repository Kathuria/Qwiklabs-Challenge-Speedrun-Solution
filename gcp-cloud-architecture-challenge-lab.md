# Cloud Architecture: Challenge Lab Solution
## GSP314

### Challenge scenario
Some Jooli Inc. standards you should follow:
- Create all resources in the `us-east1` region and `us-east1-b` zone, unless otherwise directed.
- Use the project VPCs.
- Naming is normally team-resource, e.g. an instance could be named **kraken-webserver1**.
- Allocate cost effective resource sizes. Projects are monitored and excessive resource use will result in the containing project's termination (and possibly yours), so beware. This is the guidance the monitoring team is willing to share; unless directed, use `n1-standard-1`.

  **Your region and zone maybe different**

```
gcloud config set compute/zone us-east1-b
gcloud config set project [GCP Project ID]
```

### Task 1: Create the production environment
The previous Cloud Architect had written the **Deployment Manager** configuration to build the network for kraken's production environment. You can find the DM configuration on your **jumphost** in `/work/dm`. Create the network using the Deployment Manager configuration (`/work/dm/prod-network.yaml` and `/work/dm/prod-network.jinja`). Make sure you review the configuration before deploying it.

```
cat >> prod-network.jinja << EOF
resources:
{# Network #}
- name: kraken-prod-vpc
  type: gcp-types/compute-v1:networks
  properties:
    description: "Kraken Production VPC"
    autoCreateSubnetworks: false

{# Subnet #}
- name: kraken-prod-subnet
  type: gcp-types/compute-v1:subnetworks
  properties:
    ipCidrRange: 192.168.12.0/24
    network: \$(ref.kraken-prod-vpc.selfLink)
    region: {{ properties['region'] }}
    privateIpGoogleAccess: true
    enableFlowLogs: true
    logConfig:
      enable: true
      flowSampling: 1

{# Firewall Rule #}
- name: kraken-prod-fw-ssh
  type: gcp-types/compute-v1:firewalls
  properties:
    network: projects/{{ env['project'] }}/global/networks/kraken-prod-vpc
    sourceRanges: [35.235.240.0/20]
    priority: 100
    allowed:
    - IPProtocol: tcp
      ports: [22]
    targetTags:
    - ssh-ingress
  metadata:
    dependsOn: 
    - kraken-prod-vpc
EOF

cat >> prod-network.yaml << EOF
imports:
- path: prod-network.jinja

resources:
- name: prod-network
  type: prod-network.jinja
  properties:
    region: us-east1
EOF
```

You will also need to create the Kubernetes environment. The application is already created and in the Container Repository.
- Create a two (2) node cluster called `kraken-prod` in the `kraken-prod-vpc` (remember to use `--num-nodes` to create 2 nodes only).

```
gcloud deployment-manager deployments create advanced-configuration --config prod-network.yaml
gcloud container clusters create kraken-prod --num-nodes 2 --zone us-east1-b --network kraken-prod-vpc --subnetwork kraken-prod-subnet
gcloud container clusters get-credentials kraken-prod --zone=us-east1-b
```

- Use `kubectl` with the files in `/work/k8s` to create the frontend and backend deployments and services (which will expose the frontend service via a load balancer).

```
cat >> deployment-prod-backend.yaml << EOF
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: sample-backend-production
spec:
  replicas: 2
  template:
    metadata:
      name: backend
      labels:
        app: sample
        role: backend
        env: production
    spec:
      containers:
      - name: backend
        image: gcr.io/qwiklabs-resources/sample-app:v1.0.0
        resources:
          limits:
            memory: "500Mi"
            cpu: "100m"
        imagePullPolicy: Always
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
        env:
        - name: COMPONENT
          value: backend
        - name: VERSION
          value: production
        ports:
        - name: backend
          containerPort: 8080
EOF
          
cat >> service-prod-backend.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: sample-backend-production
  name: sample-backend-production
spec:
  ports:
  - name: 8080-8080
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: sample
    role: backend
  sessionAffinity: None
  type: ClusterIP
EOF

cat >> deployment-prod-frontend.yaml << EOF
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: sample-frontend-production
spec:
  replicas: 2
  template:
    metadata:
      name: frontend
      labels:
        app: sample
        role: frontend
        env: production
    spec:
      containers:
      - name: frontend
        image: gcr.io/qwiklabs-resources/sample-app:v1.0.0
        resources:
          limits:
            memory: "500Mi"
            cpu: "100m"
        imagePullPolicy: Always
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
        env:
        - name: COMPONENT
          value: frontend
        - name: BACKEND_URL
          value: http://sample-backend-production:8080/metadata
        - name: VERSION
          value: production
        ports:
        - name: frontend
          containerPort: 8080
EOF

cat >> service-prod-frontend.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: sample-frontend-production
  name: sample-frontend-production
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: sample
    role: frontend
  sessionAffinity: None
  type: LoadBalancer
EOF
```

```
kubectl create -f deployment-prod-backend.yaml
kubectl create -f deployment-prod-frontend.yaml
kubectl create -f service-prod-backend.yaml
kubectl create -f service-prod-frontend.yaml
```

### Task 2: Setup the Admin instance
You need to set up an admin machine for the team to use.
- Once you create the **kraken-prod-vpc**, you will need to add an instance called `kraken-admin`, a network interface in **kraken-mgmt-subnet** and another in **kraken-prod-subnet**.

```
gcloud compute instances create kraken-admin --zone=us-east1-b --machine-type=n1-standard-1 --network-interface subnet=kraken-mgmt-subnet --network-interface subnet=kraken-prod-subnet
```

- You need to monitor **kraken-admin** and if **CPU utilization** is over **50%** for more than a minute you need to send an email to yourself, as admin of the system.

At Google Cloud Platform, navigate to **Monitoring**, and wait workspace to create.
```
export EMAIL=$(gcloud auth list --filter=status:ACTIVE --format="value(account)")
cat >> channel.json << EOF
{
      "type": "email",
      "displayName": "Alert notifications",
      "description": "An address to send email",
      "labels": {
        "email_address": "$EMAIL"
      },
}
EOF

gcloud beta monitoring channels create --channel-content-from-file="channel.json"
```
```
export CHANNEL=$(gcloud beta monitoring channels list --format="value(name)")
cat >> alert.json << EOF
{
   "combiner":"OR",
   "conditions":[
      {
         "conditionThreshold":{
            "aggregations":[
               {
                  "alignmentPeriod":"60s",
                  "perSeriesAligner":"ALIGN_MEAN"
               }
            ],
            "comparison":"COMPARISON_GT",
            "duration":"60s",
            "filter":"metric.type=\"compute.googleapis.com/instance/cpu/utilization\" resource.type=\"gce_instance\" metric.label.\"instance_name\"=\"kraken-admin\"",
            "thresholdValue":0.5,
            "trigger":{
               "count":1
            }
         },
         "displayName":"GCE VM Instance - CPU utilization for kraken-admin"
      }
   ],
   "displayName":"kraken-admin",
   "enabled":true,
   "notificationChannels":[
      "$CHANNEL"
   ]
}
EOF

gcloud alpha monitoring policies create --policy-from-file="alert.json"

```

### Task 3: Verify the Spinnaker deployment
The previous architect set up Spinnaker in **kraken-build-vpc**. Please connect to the Spinnaker console and verify that the built resources are working.

- To access the Spinnaker console use Cloud Shell and `kubectl` to **port forward** the **spin-deck** pod from port **9000** to **8080** and then use Cloud Shell's **web preview**.

```
gcloud container clusters get-credentials spinnaker-tutorial --region=us-east1-b
export DECK_POD=$(kubectl get pods --namespace default -l "cluster=spin-deck" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward --namespace default $DECK_POD 8080:9000 >> /dev/null &
```

- You must test that a change to the source code will result in the automated deployment of the new build. You should pull the **sample-app** repository to make the changes. Make sure you push a **new, updated, tag**.

```
gcloud source repos clone sample-app
cd sample-app
git config --global user.email "$(gcloud config get-value core/account)"
git config --global user.name "$(whoami)"
export PROJECT=$(gcloud info --format='value(config.project)')
sed -i s/PROJECT/$PROJECT/g k8s/deployments/*
git add .
git commit -a -m "Set project ID"
git tag v2.0.0
git push --tags
```
At Google Cloud Platform, navigate to **Cloud Build > History**, a new build is running.

### Check all progress, if lucky, you got the **Badge**!