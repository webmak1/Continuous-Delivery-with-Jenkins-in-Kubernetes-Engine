# [DevOps] [GSP051] Continuous Delivery with Jenkins in Kubernetes Engine

Lab:  
https://www.qwiklabs.com/focuses/1104?parent=catalog

Promo - 1 month free:  
https://go.qwiklabs.com/qwiklabs-free


Much Better doc:  
https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes


<br/>

### Clone the repository

    $ gcloud config set compute/zone us-east1-d
    $ git clone https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes.git
    $ cd continuous-deployment-on-kubernetes

<br/>

### Provisioning Jenkins


    $ gcloud container clusters create jenkins-cd \
    --num-nodes 2 \
    --machine-type n1-standard-2 \
    --scopes "https://www.googleapis.com/auth/source.read_write,cloud-platform"


<br/>

    $ gcloud container clusters list
    NAME        LOCATION    MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION    NUM_NODES  STATUS
    jenkins-cd  us-east1-d  1.14.10-gke.27  35.237.28.115  n1-standard-2  1.14.10-gke.27  2          RUNNING

<br/>

    $ gcloud container clusters get-credentials jenkins-cd
    $ kubectl cluster-info


<br/>

### Install Helm


    $ wget https://storage.googleapis.com/kubernetes-helm/helm-v2.14.1-linux-amd64.tar.gz

    $ tar zxfv helm-v2.14.1-linux-amd64.tar.gz
    $ cp linux-amd64/helm .

<br/>

// Add yourself as a cluster administrator in the cluster's RBAC so that you can give Jenkins permissions in the cluster:

    $ kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)

<br/>

// Grant Tiller, the server side of Helm, the cluster-admin role in your cluster:

    $ kubectl create serviceaccount tiller --namespace kube-system

    $ kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller


// Initialize Helm. This ensures that the server side of Helm (Tiller) is properly installed in your cluster.

    $ ./helm init --service-account=tiller
    $ ./helm update

    $ ./helm version
    Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
    Server: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}

<br/>

### Configure and Install Jenkins

    $ ./helm install -n cd stable/jenkins -f jenkins/values.yaml --version 1.2.2 --wait

    $ kubectl get pods
    NAME                          READY   STATUS    RESTARTS   AGE
    cd-jenkins-57f697956c-kj2n9   1/1     Running   0          3m4s

<br/>
    // Configure the Jenkins service account to be able to deploy to the cluster.

    $ kubectl create clusterrolebinding jenkins-deploy --clusterrole=cluster-admin --serviceaccount=default:cd-jenkins

<br/>

// Run the following command to setup port forwarding to the Jenkins UI from the Cloud Shell

    $export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")

<br/>

    $ kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &

<br/>

    $ kubectl get svc
    NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
    cd-jenkins         ClusterIP   10.35.250.193   <none>        8080/TCP    5m49s
    cd-jenkins-agent   ClusterIP   10.35.240.194   <none>        50000/TCP   5m49s
    kubernetes         ClusterIP   10.35.240.1     <none>        443/TCP     13m



You are using the Kubernetes Plugin so that our builder nodes will be automatically launched as necessary when the Jenkins master requests them. Upon completion of their work, they will automatically be turned down and their resources added back to the clusters resource pool.

Notice that this service exposes ports 8080 and 50000 for any pods that match the selector. This will expose the Jenkins web UI and builder/agent registration ports within the Kubernetes cluster. Additionally, the jenkins-ui services is exposed using a ClusterIP so that it is not accessible from outside the cluster.

<br/>

### Connect to Jenkins

    // The Jenkins chart will automatically create an admin password for you. To retrieve it, run:

    $ printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo


To get to the Jenkins user interface, click on the Web Preview button in cloud shell, then click “Preview on port 8080”:

admin/<YOUR_PASSWORD>

<br/>

![Application](/img/pic1.png?raw=true)

<br/>

### Deploying the Application

You will deploy the application into two different environments:

* Production: The live site that your users access.
* Canary: A smaller-capacity site that receives only a percentage of your user traffic. Use this environment to validate your software with live traffic before it's released to all of your users.

<br/>

    $ cd sample-app

    // Create the Kubernetes namespace to logically isolate the deployment:
    $ kubectl create ns production

<br/>

    $ kubectl apply -f k8s/production -n production
    $ kubectl apply -f k8s/canary -n production
    $ kubectl apply -f k8s/services -n production

<br/>

    $ kubectl scale deployment gceme-frontend-production -n production --replicas 4

<br/> 

    $ kubectl get pods -n production -l app=gceme -l role=frontend
    NAME                                         READY   STATUS    RESTARTS   AGE
    gceme-frontend-canary-7c89746787-rh64c       1/1     Running   0          60s
    gceme-frontend-production-867b7cbf98-2gs7d   1/1     Running   0          26s
    gceme-frontend-production-867b7cbf98-gmhjh   1/1     Running   0          67s
    gceme-frontend-production-867b7cbf98-jwdtr   1/1     Running   0          26s
    gceme-frontend-production-867b7cbf98-rxjsc   1/1     Running   0          26s

<br/>


    $ kubectl get pods -n production -l app=gceme -l role=backend
    NAME                                      READY   STATUS    RESTARTS   AGE
    gceme-backend-canary-df5559d77-hxbvv      1/1     Running   0          2m26s
    gceme-backend-production-cc949994-wlshw   1/1     Running   0          2m32s

<br/>

It can take several minutes before you see the load balancer external IP address.

<br/>

    $ for i in `seq 1 5`;do kubectl --namespace=production get service gceme-frontend; sleep 60;done

<br/>

    $ kubectl get service gceme-frontend -n production
    NAME             TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
    gceme-frontend   LoadBalancer   10.35.249.249   35.231.32.131   80:30285/TCP   2m43s

<br/>

    $ export FRONTEND_SERVICE_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)

<br/>

    $ curl http://$FRONTEND_SERVICE_IP/version
    1.0.0


<br/>

![Application](/img/pic2.png?raw=true)

<br/>

### Creating the Jenkins Pipeline

// Create a copy of the gceme sample app and push it to a Cloud Source Repository:

    $ gcloud source repos create default
    $ git init

    // Initialize the sample-app directory as its own Git repository:
    $ git config credential.helper gcloud.sh

    $ git remote add origin https://source.developers.google.com/p/$DEVSHELL_PROJECT_ID/r/default

    $ git config --global user.email "marley@example.com"
    $ git config --global user.name "marley"

    $ git add .
    $ git commit -m "Initial commit"
    $ git push origin master

<br/>

#### Adding your service account credentials


    Jenkins  --> Credentials --> Jenkins --> Global credentials (unrestricted) --> Add Credentials


Google Service Account from metadata

<br/>

![Application](/img/pic3.png?raw=true)

<br/>

![Application](/img/pic4.png?raw=true)

<br/>

#### Creating the Jenkins job

Jenkins > New Item

    Name: sample-app
    Multibranch Pipeline option
    OK.

<br/>

Step 3: On the next page, in the Branch Sources section, click Add Source and select git.

Step 4: Paste the HTTPS clone URL of your sample-app repo in Cloud Source Repositories into the Project Repository field. Replace [PROJECT_ID] with your GCP Project ID:

https://source.developers.google.com/p/qwiklabs-gcp-00-5d7bec341228/r/default


<br/>

![Application](/img/pic5.png?raw=true)

Step 5: From the Credentials drop-down, select the name of the credentials you created when adding your service account in the previous steps.


<br/>

![Application](/img/pic6.png?raw=true)

Step 6: Under Scan Multibranch Pipeline Triggers section, check the Periodically if not otherwise run box and set the Interval value to 1 minute.

Step 7: Your job configuration should look like this:

Step 8: Click Save leaving all other options with their defaults

After you complete these steps, a job named "Branch indexing" runs. This meta-job identifies the branches in your repository and ensures changes haven't occurred in existing branches. If you click sample-app in the top left, the master job should be seen.

    Note: The first run of the master job might fail until you make a few code changes in the next step.

You have successfully created a Jenkins pipeline! Next, you'll create the development environment for continuous integration.

<br/>

![Application](/img/pic7.png?raw=true)


<br/>

### Creating the Development Environment

    $ git checkout -b new-feature

<br/>


    $ vi Jenkinsfile

REPLACE_WITH_YOUR_PROJECT_ID - qwiklabs-gcp-00-5d7bec341228


<br/>

    $ vi html.go

replace

    <div class="card blue">

on

    <div class="card red">

<br/>

    $ vi main.go

<br/>

    const version string = "1.0.0"

on

    const version string = "2.0.0"


<br/>

### Kick off Deployment

    $ git add Jenkinsfile html.go main.go

    $ git commit -m "Version 2.0.0"

    $ git push origin new-feature

After the change is pushed to the Git repository, navigate to the Jenkins user interface where you can see that your build started for the new-feature branch. It can take up to a minute for the changes to be picked up.


<br/>

    $ kubectl proxy &

    $ curl \
    http://localhost:8001/api/v1/namespaces/new-feature/services/gceme-frontend:80/proxy/version

    2.0.0

<br/>

### Deploying a Canary Release


    $ git checkout -b canary
    $ git push origin canary

    $ export FRONTEND_SERVICE_IP=$(kubectl get -o \
    jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)

    $ while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0
    1.0.0

<br/>

### Deploying to production

    $ git checkout master

    $ git merge canary

    $ git push origin master

    $ export FRONTEND_SERVICE_IP=$(kubectl get -o \
    jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)

    $ while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done
    1.0.0
    1.0.0
    1.0.0
    2.0.0
    2.0.0
    1.0.0
    1.0.0
    1.0.0
    2.0.0
    1.0.0

<br/>

    $ kubectl get service gceme-frontend -n production


<br/>

![Application](/img/pic8.png?raw=true)


<br/>

![Application](/img/pic9.png?raw=true)

---



<a href="https://marley.org"><strong>Marley</strong></a>
