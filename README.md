# skjenkins
### Scalable Jenkins with Kubernetes cluster example

This setup can serve as a PoC for creating a scalable Jenkins deployment on a Kubernetes cluster.

# Prerequisites
1. A working 3 node Kubernetes cluster. (This will probably work with minikube as well, however I haven't tested this with Minikube since I have my own Kubernetes cluster)
2. Automated storage provisioner like [this one](https://github.com/spirit986/external-storage/tree/master/nfs). Else you will need to modify the appropriate parts from `jenkins-deployment.yaml` and setup your own storage provisioner or just use an `emptyDir` volume.

# Usage example
A short tutorial on how to use the scripts.

# Deploying the Jenkins Master

## Get your kube master IP info
Bellow example is for my own kube master and I will use this IP throughout the example document.
```bash
$ kubectl cluster-info | grep master
Kubernetes master is running at https://172.16.0.200:6443
```

## Build the Jenkins Dockerfile
Use the provided Jenkins Dockerfile. This will be our Jenkins master. I am using the jenkins/jenkins:lts image from the official Dockerhub repo. Once you build the docker image push it as your own in Dockerhub.

**Example:**
```bash
$ sudo docker build -t spirit986/spirit-jenkins:latest
$ sudo docker push spirit986/spirit-jenkins:latest
```

## Create the Jenkins service account
Create the Jenkins service account, the ClusterRole and the ClusterRoleBinding in Kubernetes.
```bash
## First create the service account itself
$ kubectl create serviceaccount jenkins

## Use the jenkins-access-role-binding.yaml to create the CR and the CRB
$ kubectl create -f create -f jenkins-access-role-binding.yaml
```

### Optional step 
### Check the details of the service account and verify it works:
```bash
$ kubectl get serviceaccount jenkins -o json | jq -Mr '.secrets[].name'
jenkins-token-phshv

$ kubectl get secrets jenkins-token-phshv -o json | jq -Mr '.data.token' | base64 --decode
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImplbmtpbnMtdG9rZW4tcGhzaHYiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiamVua2lucyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImY3N2I5ZjFhLTliZGQtMTFlOS1hMzY5LTA4MDAyNzVjNGNjZiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmplbmtpbnMifQ.MnuvxtD12jlsQYxKjtgWss9LKUu6wCjuPUtojbQ1hDsqJEytSHPig_mV1KEs3AUlgbUqol4OxUmMJax8kbSH3XnYiko-4w9bwyvPVATw7XLQFwq7bJ6Ua6z8Uekr3sQ-nxKTjTJpfVtMjf1ymeRS5tjDZZg88y1vGZXsNmjBDSflPUjUOy4EKv_HFZUu6H1XGKIZXj4TZb1vCvPBqQEBsCxcyVxa0HbgEMjntjjvtAaZVUpqPfz5BjCUvKkN6Ctm2qwy97V7runtqxYjEmloiRYIMZzItfCLEpWEYqz6pNQhXs6oyJ4RkiaByblWULEzXnunOHLp4YyGn6m9cU7gtA

## This checks that your service account works
$ curl -k https://172.16.0.200:6443/api/v1/namespaces -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImplbmtpbnMtdG9rZW4tcGhzaHYiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiamVua2lucyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImY3N2I5ZjFhLTliZGQtMTFlOS1hMzY5LTA4MDAyNzVjNGNjZiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmplbmtpbnMifQ.MnuvxtD12jlsQYxKjtgWss9LKUu6wCjuPUtojbQ1hDsqJEytSHPig_mV1KEs3AUlgbUqol4OxUmMJax8kbSH3XnYiko-4w9bwyvPVATw7XLQFwq7bJ6Ua6z8Uekr3sQ-nxKTjTJpfVtMjf1ymeRS5tjDZZg88y1vGZXsNmjBDSflPUjUOy4EKv_HFZUu6H1XGKIZXj4TZb1vCvPBqQEBsCxcyVxa0HbgEMjntjjvtAaZVUpqPfz5BjCUvKkN6Ctm2qwy97V7runtqxYjEmloiRYIMZzItfCLEpWEYqz6pNQhXs6oyJ4RkiaByblWULEzXnunOHLp4YyGn6m9cU7gtA"
```

## Build the jenkins master
Use the provided `jenkins-deployment.yaml` to build the master jenkins container. Feel free to modify any parts of the deployment script according to your specific enviorment.
```bash
$ kubectl apply -f jenkins-deployment.yaml
```

## Inspect your jenkins setup
To inspect:
```bash
$ kubectl get service | grep jenkins
jenkins                        NodePort       10.111.141.48   <none>         8080:30001/TCP                       28h
```

Since my kube master IP is 172.16.0.200 I should be able to open Jenkins on: http://172.16.0.200:30001.

# Enabling the Jenkins slaves
In the main dockerfile we installed the Kubernetes plugin. To enable and test the slaves you can now go into ***Manage Jenkins -> Configure System***, and near the bottom there is the ***Cloud*** section with the Kubernetes option.

Under the Kubernetes section you will need to update:
* Name - Anything you want to distinguish your Kubernetes cluster, since more than one cluster can be added.
* Kubernetes URL - In my example this URL is: https://172.16.0.200:6443 as shown from the output of the command at the beggining.
* Jenkins URL - To get the jenkins URL you can simply put the HTTP://IP:PORT URL of the pod running jenkins. To get the IP of the pod `$ kubectl get pods -o wide | grep jenkins`. In my case the URL is: http://10.244.2.23:8080.

As a next step you will also need to update the ***Kubernetes Pod Template***. The relevant fields are marked bellow:
* Name - I used jenkins-slave here. All of the jenkins slave containers will bear the jenkins-slave name to them. This way you will be able to distinguish them easly.
* Labels - jenkins-slave.
* Usage - Use this node as much as possible.

***Container Template***
* Name - jenkins-slave.
* Docker Image - jenkins/jnlp-slave - This is the official jenkins slave docker image from DockerHub.

Apply and save the configuration.

### Optional step
Under ***Manage Jenkins -> Manage Nodes*** section, edit the `master` node and configure the following options:
* # of executors	- Set this to 1 or 0. This will inturn result jenkins to schedule the builds on the slave nodes only.

# Test the setup
If everything is done correctly you will be able to test your setup the following way.
1. Create two or three build jobs. I named mine *first* and *second*.
2. At the configuration options for each job, under the Build tab, add ***Execute shell*** build step with `sleep 60` or `sleep 90` more seconds. This way we can simulate long builds which will trigger the creation of the slave pods.
3. Schedule the Jenkins builds. 

If everything is done correctly up until this step, you should be able to see `jenkins-slave-HASHID`, jenkins pods being created in your cluster. 
```bash
$ kubectl get pods | grep jenkins
NAME                                            READY   STATUS    RESTARTS   AGE
jenkins-7576859b79-tnq77                        1/1     Running   0          4h39m
jenkins-slave-jrwrh                             2/2     Running   0          7s
```
Additionally at the ***Build executor status*** widget on the main dashboard you will also be able to see the jenkins slaves with the same name.
Example:
![Jenkins slaves](https://imgur.com/a/x7cUsV2)

