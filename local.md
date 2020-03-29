# [DevOps] [GSP051] Continuous Delivery with Jenkins in Kubernetes Engine


    $ cd ~/tmp
    $ git clone https://github.com/webmakaka/continuous-deployment-on-kubernetes
    $ cd continuous-deployment-on-kubernetes

<br/>


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
    gceme-frontend-canary-67f5ffdcdd-4khds       1/1     Running   0          2m31s
    gceme-frontend-production-5f9dc74dfc-bxkjb   1/1     Running   0          57s
    gceme-frontend-production-5f9dc74dfc-lbngs   1/1     Running   0          4m51s
    gceme-frontend-production-5f9dc74dfc-lhrbv   0/1     Pending   0          57s
    gceme-frontend-production-5f9dc74dfc-vf65v   1/1     Running   0          57s


<br/>


    $ kubectl get pods -n production -l app=gceme -l role=backend
    NAME                                        READY   STATUS    RESTARTS   AGE
    gceme-backend-canary-76f8f78894-4lhsw       1/1     Running   0          3m18s
    gceme-backend-production-5c59b5bdcc-7txbq   1/1     Running   0          5m38s


<br/>

    $ kubectl get service gceme-frontend -n production
    NAME             TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
    gceme-frontend   LoadBalancer   10.98.76.171   <pending>     80:32429/TCP   3m34

<br/>

    $ curl 192.168.0.11:32429/version
    1.0.0

<br/>

http://192.168.0.11:32429/


<br/>

<br/>

![Application](/img/pic-local-01.png?raw=true)


<br/>

    $ kubectl get deployments -n production
    $ kubectl edit deployments  gceme-frontend-production -n production

Change manually latest on 1.0.0


<br/>

### Jenkins

Install plugin

<br/>

![Application](/img/pic-local-02.png?raw=true)

https://plugins.jenkins.io/kubernetes/

https://www.youtube.com/watch?v=DAe2Md9sGNA

<br/>

Manage Jenkins --> Configure System --> Cloud 


<br/>

![Application](/img/pic-local-03.png?raw=true)


<br/>

![Application](/img/pic-local-04.png?raw=true)


<br/>

### Create a new job



---



<a href="https://marley.org"><strong>Marley</strong></a>
