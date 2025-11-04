Here, we see a step-by-step process for setting up Jenkins on a Kubernetes Cluster.

# Setup Jenkins On Kubernetes

For setting up a Jenkins Cluster on Kubernetes, we will do the following:

1. [Create a Namespace]
2. [Create a service account] with Kubernetes admin permissions.
3. [Create local persistent volume] for persistent Jenkins data on Pod restarts.
4. [Create a deployment YAML] and deploy it.
5. [Create a service YAML]and deploy it.

**Step 1** : Create a Namespace for Jenkins. It is good to categorize all the DevOps tools as a separate namespace from other applications.

```
kubectl create namespace devops-tools
```

**Step 2:** Create a 'jenkins-01-serviceAccount.yaml' file and copy the following admin service account manifest.

The 'jenkins-01-serviceAccount.yaml' creates a 'jenkins-admin' clusterRole, 'jenkins-admin' ServiceAccount and binds the 'clusterRole' to the service account.

The 'jenkins-admin' cluster role has all the permissions to manage the cluster components. You can also restrict access by specifying individual resource actions.

Now create the service account using kubectl.

```
kubectl apply -f jenkins-01-serviceAccount.yaml
```

**Step 3:** Create 'jenkins-02-volume.yaml' and copy the following persistent volume manifest.

**Important Note:** Replace 'worker-node01' with any one of your cluster worker nodes hostname. I have used node01.

You can get the worker node hostname using the kubectl.

```
kubectl get nodes
```

**For volume**, we are using the '*local*' StorageClass for the purpose of demonstration. Meaning, it creates a 'PersistentVolume' volume in a specific node under the '/mnt' location.

*As the 'local' storage class requires the node selector, you need to specify the worker node name correctly for the Jenkins pod to get scheduled in the specific node.*

If the pod gets deleted or restarted, the data will get persisted in the node volume. However, if the node gets deleted, you will lose all the data.

Ideally, you should use a persistent volume using the available storage class with the cloud provider, or the one provided by the cluster administrator to persist data on node failures.

Let’s create the volume using kubectl

```
kubectl create -f jenkins-02-volume.yaml
```

**Step 4:** Create a Deployment file named 'jenkins-03-deployment.yaml' and copy the following deployment manifest.

Create the deployment using kubectl.

```
kubectl apply -f jenkins-03-deployment.yaml
```

Check the deployment status.

```
kubectl get deployments -n devops-tools
```

Now, you can get the deployment details using the following command.

```
kubectl describe deployments --namespace=devops-tools
```

### Accessing Jenkins Using Kubernetes Service

We have now created a deployment. However, it is not accessible to the outside world. For accessing the Jenkins deployment from the outside world, we need to create a service and map it to the deployment.

Create 'jenkins-04-service.yaml'

Here, we are using the type as 'NodePort' which will expose Jenkins on all kubernetes node IPs on port 32000. If you have an ingress setup, you can create an ingress rule to access Jenkins. Also, you can expose the Jenkins service as a Loadbalancer if you are running the cluster on AWS, Google, or Azure cloud.

Create the Jenkins service using kubectl.

```
kubectl apply -f jenkins-04-service.yaml
```

Now, when browsing to any one of the Node IPs on port 32000, you will be able to access the Jenkins dashboard. You can see the node ip executing kubectl get nodes -o wide

```
http://<node-ip>:32000 
```

Jenkins will ask for the initial Admin password when you access the dashboard for the first time.

You can get that from the pod logs either from the Kubernetes dashboard or CLI. You can get the pod details using the following CLI command.

```
kubectl get pods --namespace=devops-tools
```

With the pod name, you can get the logs as shown below. Replace the pod name with your pod name.

```
kubectl logs jenkins-deployment-2539456353-j00w5 --namespace=devops-tools
```

Jenkins will ask for the initial Admin password when you access the dashboard for the first time.

You can get that from the pod logs either from the Kubernetes dashboard or CLI. You can get the pod details using the following CLI command.

```
kubectl get pods --namespace=devops-tools
```

With the pod name, you can get the logs as shown below. Replace the pod name with your pod name.

```
kubectl logs jenkins-5874c666f4-cgs9t --namespace=devops-tools
```

Alternatively, you can run the exec command to get the password directly from the location as shown below.

* Using the first (normally only) instance of the application pod:
  ```
  kubectl exec -it "deployment.apps/jenkins" cat /var/jenkins_home/secrets/initialAdminPassword -n devops-tools
  ```
* …or, using a specific container instance:
  ```
  kubectl exec -it jenkins-5874c666f4-cgs9t cat /var/jenkins_home/secrets/initialAdminPassword -n devops-tools
  ```

Once you enter the password, proceed to install the suggested plugins and create an admin user. All of these steps are self-explanatory from the Jenkins dashboard.

### Access jenkins using LoadBalancer(Nginx)

I have installed nginx on a separate Ubuntu machine. This will be acting as LoadBalacer.

why you want to have a LoadBalancer ?  Rightnow, you are are accessing Jenkins using any Node's IP from your cluster, which is not a best practice or optimal way to access an application.

### **1. Goal**

You want to be able to point your browser to the Ubuntu machine running Nginx (say `192.168.1.222`), and Nginx will distribute requests to Jenkins running on the Kubernetes nodes via their NodePort.

##### **2. Steps**

### **Step A: Install Nginx**

On your Ubuntu 22.04 server:

```bash
sudo apt update sudo apt install nginx -y
```

### **Step B: Nginx Load Balancer Configuration**

We’ll configure Nginx as a **reverse proxy + load balancer** to Jenkins.

Create a new config file under `/etc/nginx/conf.d/jenkins.conf`:

```conf
upstream jenkins_backend {
    ip_hash;  # optional for sticky sessions
    server 192.168.1.150:32000;
    server 192.168.1.151:32000;
    server 192.168.1.152:32000;
    server 192.168.1.153:32000;
}

server {
    listen 80;
    server_name 192.168.1.222; #This is your nginx server IP, you can use domain name if you have

    location / {
        proxy_pass http://jenkins_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        proxy_http_version 1.1;
        proxy_request_buffering off;
    }
}
```

### **Step C: Test and reload the configuration**

```

sudo nginx -t
sudo systemctl restart nginx
```

### **Step D: Access Jenkins from browser**

Now, from any machine in the same network:

```browse
http://192.168.1.222
```
