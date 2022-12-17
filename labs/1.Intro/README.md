# Lab : Core Kubernetes Concepts

> Estimated Duration: 60 minutes

## Module 1 Table of Contents

[Exercise: Setup your Azure subscription](#exercise-create-a-basic-azure-kubernetes-service-aks-cluster)

[Exercise: Creating a Pod Declaratively](#exercise-creating-a-pod-declaratively)

[Exercise: Working with Deployments](#exercise-working-with-deployments)

[Exercise: Working with Services](#exercise-working-with-services)

[Exercise: Cleanup](#exercise-cleanup)


# Exercise: Setup your Azure subscription

In this exercise you will create a simple AKS cluster.  In the next module, you'll create a more complete one.



### Task 1 - Login into your subscription and register providers

1. Log in to your Azure subscription at https://portal.azure.com.
1. At the top of the portal window, click on the Cloud Shell icon.

![](content/cloudshell.png)

3. Make sure that the Cloud Shell window is open in the Bash mode.

   ![](content/bash.png)

4. At this point, you are logged in to Azure automatically via Cloud Shell, but you need to make sure Cloud Shell is pointed to the correct subscription if you have multiple Azure subscriptions. Set the current subscription.Set the current subscription.

```PowerShell
az account set --subscription "Azure Pass - Sponsorship"
```

5. Register needed providers. It may take a few minutes for some of the providers to finish registering.

```PowerShell
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Monitor
az provider register --namespace Microsoft.ManagedIdentity
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.OperationsManagement
az provider register --namespace Microsoft.KeyVault
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.Kubernetes
```

6. Search for and open the **Subscriptions** blade.  Select your subscription.

7. Scroll down and select **Resource providers**.

![](content/azure-resources.png)

8. Watch the progress of the registration process until all the providers listed above have been registered.  Click the *Refresh* button every few minutes to update the progess.  Once everything has been registered, continue with the tasks in this lab.

# Exercise: Creating a Pod Declaratively

This Exercise demonstrates the use of a YAML file to create a pod declaratively.



### Task 1 - Create a Pod declaratively

1. Change into the **Module1** folder

```bash
cd labs\1.Intro
```

2. Use the YAML file provided to create a Pod.  You may want to open the **simple-pod.yaml** file and review its contents.

The pod definition contains the **Nginx** container that listens to port 80.

```bash
kubectl apply -f simple-pod.yaml
```

3. Now, make sure pod is up and running.

```bash
kubectl get pods
```

You should see a pod named **nginx-pod**

![](content/simple-pod.png)

4. Add a second pod, then check the list again.

```bash
kubectl apply -f simple-pod2.yaml
kubectl get pods
```



### Task 2 - Filter pods based on a label

1. Show all the labels in the pods

```bash
kubectl get pods --show-labels
```

![](content/pod-list-labels.png)

2. Let's say you want to list pods that have a label named  **kind=web** associated with them. You can use **-l** switch to apply filter based on labels.

```bash
kubectl get pod -l kind=web
```

3. To prove that this works as expected, run the command again but change the value of label **kind** to **db**. Notice, this time *kubectl* doesn't return any pods because there are no pods that match the label **kind** and a value of **db**.

```bash
kubectl get pod -l kind=db
```



### Task 3 - View complete definition of the Pod

1. Query Kubernetes to return the complete definition of a Pod from its internal database by exporting the output (**-o**) to **YAML**.  Then pipe the result to a file.

```bash
kubectl get pods nginx-pod -o yaml > mypod.yaml
```

> To view the JSON version, use the **-o json** flag instead.

2.  View the contents of the generated file in VS Code (or an editor of your choice).

```bash
code mypod.yaml
```
![](content/pod-details.png)

**NOTE:** Observe all the properties that Kubernetes populated with default values when it saved the Pod definition to its database.

### Task 4 - Delete the Pods

1. Delete the Pods that were created in this exercise.

```bash
kubectl delete pod nginx-pod
kubectl delete pod nginx-pod2
```

# Exercise: Working with Deployments

In this Exercise, you will create a Deployment and rollout an application update.  Deployments provide a consistent mechanism to upgrade an application to a new version, while keeping the downtime to a minimum.  Note that internally, Deployments use *ReplicaSets* for managing Pods.  However, you never work directly with *ReplicaSets* since Deployments abstract out that interaction.



### Task 1 - Create a new Deployment

The **ng-dep.yaml** file contains a Deployment manifest.  The Pod in the *template* contains an *nginx* container with a tag **1.0**.  The **1.0** represents the version of this container and hence of the application running inside it.

1. Create a Deployment and a Service to access the Pods of the deployment.

```bash
kubectl apply -f ng-dep.yaml
kubectl apply -f ng-svc.yaml
```

**NOTE:** The *--record* flag saves the command you applied in the deployment's ReplicaSet history.  This helps in deciding which previous Revision to roll back to if needed.


2. Run the following command to see the Pods, ReplicaSets, Deployments and Services that were created.

```bash
kubectl get all --show-labels
```
![](content/get-all.png)



### Task 2 - Access version 1.0 of application

1. Wait about 3-4 minutes to allow Azure to create a Public IP address for the service.  Check to see if an address has been assigned by getting the list of services.

```bash
kubectl get svc
```

![](content/services.png)


2. When you see an **EXTERNAL-IP** assigned, open a browser with that address.  Example: **http://20.81.24.216**

![](content/kube1.png)



### Task 3 - Update the Deployment to version 2.0

You are now going to update the Deployment to use version **2.0** of the container instead of **1.0**.  This can be done in one of two ways. One approach is to use *imperative* syntax, which is faster and is often used during the development/testing stage of an application.  The alternate method is to update the YAML file and then to reapply it to the cluster.

1. To start rolling out the new update, change the container image tag from **1.0** to **2.0** by running this command:

```bash
kubectl set image deployment ng-dep nginx=k8slab/nginx:2.0
```

2. In the command above, **ng-dep** is the name of Deployment and **nginx** is the name of the container within the Pod template.  The change will force the Deployment to create a new ReplicaSet with an image tagged **2.0**.

3. List all the pods and notice that old pods are terminating and that new Pods have been created.

```bash
kubectl get pods
```

4. Run the follwing command to review the Deployment definition with the updated value of container image:

```bash
kubectl describe deployment ng-dep
```

![](content/kube-describe.png)

> Notice the Image section (under Containers) shows the value of container image as **2.0**.

5. Run the command to view the Pods, ReplicaSets and Deployments again.

```bash
kubectl get all
```

![](content/get-all-2.png)

> Notice that the old replica set still exists, even though it has 0 Desired Pods.

6. Run the *describe* command on that old ReplicaSet.

```bash
kubectl describe rs <old replicaset name>
```

![](content/old-rs.png)

> Notice that the old definition still has the previous version number.  This is maintained so you can roll back the change to that version if you which.

7. Access the 2.0 version of application by refreshing the browser at the same address as above.

![](content/kube2.png)



### Task 4 - Rollback the Deployment

The purpose of maintaining the previous **ReplicaSet** is to be able to rollback changes to any previous version.

1. Review the deployment history.

```bash
kubectl rollout history deploy/ng-dep
```

2. Rollback the Deployment to the previous version.

```bash
kubectl rollout undo deploy/ng-dep
```

3. Wait a few seconds and refresh the browser again.

![](content/kube1.png)

> Notice the site is back to the previous version.



### Task 5 - Delete the Deployment and Service

1. Delete the Deployment and Service

```bash
kubectl delete deployment ng-dep
kubectl delete service ng-svc
```

**NOTE:** It may take a few minutes to delete the service because has to delete the Public IP resource in Azure.


# Exercise: Working with Services

In this Exercise you will create a simple Service.  Services help you expose Pods externally using label selectors.



### Task 1 - Create a new Service

1. Create a deployment.

```bash
kubectl apply -f sample-dep.yaml
```

2. The **sample-svc.yaml** file contains a Service manifest.  Services use label selectors to determine which Pods it needs to track and forward the traffic to.

1. Review running Pods and their labels.

```bash
kubectl get pods --show-labels
```

> Notice the label **sample=color** that is associated with the Pods.

2. Open the **sample-svc.yaml** file and examine the **selector** attribute.  Notice the **sample: color** selector.  This Service will track all Pods that have a label **sample=color** and load balance traffic between them.

3. Create the Service.

```bash
kubectl apply -f sample-svc.yaml
```

4. Check the of newly created service.

```PowerShell
kubectl get svc -o wide
```

The command above will display the details of all available services along with their label selectors.  You should see the **sample-svc** Service with **PORTS 80:30101/TCP** and **SELECTOR sample=color**.



### Task 2 - Access the sample-svc Service

1. Open a browser and navigate to the IP address shown in the output of the previous command.

![](content/sample-svc.png)

2. The website displays the Node IP/Pod IP address of the pod currently receiving the traffic through the service's load balancer.  The page refreshes every 3 seconds and each request may be directed to a different pod, with a different IP address.   This is the service's internal load balancer at work.



### Task 3 - Delete the Deployment and Service

Deleting any Pod will simply tell Kubernetes that the Deployment is not in its *desired* state and it will create a replacement.  You can only delete Pods by deleting the Deployment.

1. Delete the Deployment.

```bash
kubectl delete deployment sample-dep
```

2. The Service is independent of the Pods it services, so it's not affected when the Deployment is deleted.  Anyone trying to access the service's address will simply get a 404 error.  If the Deployment is ever re-created, the Service will automatically start sending traffic to the new Pods.

3.  Delete the Service.

```bash
kubectl delete service sample-svc
```



# Exercise: Cleanup


### Task 1 - Delete the rest of the Deployment(s), Service(s) and Pods



