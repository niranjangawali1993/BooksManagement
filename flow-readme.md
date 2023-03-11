# Below is the temporary document

1>
zone - asia-south1-a

- gcloud config set project <project-id> (books-management-kubernetes )
- gcloud config set compute/region <COMPUTE_REGION> (asia-south1)
- gcloud services enable container.googleapis.com (Enable the kubernetes API)
  For this step there is observed an error about billing account link, so steps to link billing accounts are added.

There are two ways we can create a cluster autopilot and standard
For standard we have to define the resources that will be required for the cluster and for auto pilot
Google manages your cluster configuration, including your nodes, scaling, security, and other preconfigured settings.

For this demo we are going with auto-pilot mode.

- gcloud container clusters create-auto mysql-demo --region=asia-south1

- We can check if the cluster is ready or not using
- gcloud container clusters list

We get the following output:

NAME: book-management
LOCATION: asia-south1
MASTER_VERSION: 1.24.9-gke.3200
MASTER_IP: 34.100.173.145
MACHINE_TYPE: e2-medium
NODE_VERSION: 1.24.9-gke.3200
NUM_NODES:
STATUS: PROVISIONING

Once the status is running we can proceed with further part

- gcloud container clusters get-credentials book-management --region asia-south1 --project mysql-kubernetes-demo

We can check the working nodes uing following command

- kubectl get nodes

- Create a namespace for development.

kubectl create namespace development

kubectl config set-context --current --namespace=development

- Create a new folder named `mysql`
  mkdir mysql

- Once folder is created we need to add the required files to it, for this purpose you may use the editor provided by gcp or command shell with vim editor as per your preference.

- Create mysql secret to store the root password.
  `
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  labels:
    name: mysql-secret
    app: mysql-gke-deployment
type: Opaque
data:
  root-password: d2ludGVy`

  kubectl create -f mysql-secret.yaml

  `
gawaliniranjan@cloudshell:~/mysql (books-management-kubernetes)$ kubectl get secrets
NAME           TYPE     DATA   AGE
mysql-secret   Opaque   1      35s`

- Create a persistent volume , so that we can retaim that msyql data even after container is destroyed and new container is created.

  kubectl create -f mysql-persistent-volume-claim.yaml

`
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-store
  labels:
    name: mysql-data-store
    app: mysql-gke-deployment
spec:
  resources:
    requests:
      storage: 3Gi
    limits:
      storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: standard-rwo`

Persistent volume is automatically created for the persistent volume claim.

- Creating configmap to store the mysql queries that should be executed while creating the pod.
  mysql-config-map.yaml
  `
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-preload-data-config
data:
  init.sql: |
    CREATE DATABASE IF NOT EXISTS`book-management-db`;
    USE `book-management-db`;
    DROP TABLE IF EXISTS `books`;
    CREATE TABLE `books`(
     `id`int NOT NULL AUTO_INCREMENT,
     `title`varchar(255) NOT NULL,
     `author`varchar(255) NOT NULL,
     `language`varchar(255) DEFAULT NULL,
     `created_at`datetime DEFAULT CURRENT_TIMESTAMP,
     `created_by`varchar(255) DEFAULT NULL,
     `updated_at`datetime DEFAULT CURRENT_TIMESTAMP,
     `updated_by` varchar(255) DEFAULT NULL,
      PRIMARY KEY (`id`)
    );
    insert into books(`title`,`author`,`language`) values('Harray Potter','J.K.Rowling','English');`

- Creating the deployment
  Now we can proceed with creating the deployment

Once deployment is crated we can check its status using
kubectl get deployments mysql-deploy

- Now we can create a service that will provide with a cluster IP that will be used for communication
  with application.

kubectl create -f mysql-service.yaml

kubectl get svc

`NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
mysql-service   ClusterIP   10.26.131.102   <none>        3306/TCP   8s`

Cluster IP is only accessible internally within that cluster.

Now we can check PVC status Bound.

- In order to connect with created mysql instance on the cluster we can use the following command.

kubectl exec --stdin --tty pod/mysql-deploy-9fd4c579d-tcctb -- /bin/bash
mysql -p // enter the password defined in the secrets

we can get pod name using `kubectl get pods`

- Now once we got the clusterIP of service now create the new image with tag `gke-mysql`
  Update the .env file and create a image using command `docker-compose build`
  For the values of .env file please refer env.sample file from repository

  push the image to the docker hub using following command

  `docker push niranjang2/books_management_service:gke-mysql`

  - Now lets create another folder named `book-management` for book management service configuration on the cluser.

As we did earlier please copy content of deploy and service to newly created folder on cluster.

Describe the deployment file

- Create the deployment of book mangement.
  `kubectl create -f book-mgmt-api-deploy.yaml`

Wait for deployment to be scheduled.

you can check the schedule using following command

kubectl get deploy,pods

- Now lets create service

describe service

`kubectl create -f book-mgmt-api-service.yaml`

we need the external-ip address for communicating with the application.

Once the deployment and service created successfully we can use the external ip address to communicate with
the service.

Add the deleting all configuration commands.

`kubectl delete all --all` this command deletes deployments,services,pods,replicasets

`kubectl delete persistentvolumeclaim/mysql-data-store --force` to delete created persistent volume and pvc

`kubectl delete configmap/database-config` to delete the database-config
`kubectl delete configmap/mysql-preload-data-config` to delete the mysql-preload-data-config

`kubectl delete secret/mysql-secret` to delete the mysql secret.

- Provide all the commands to check logs

- NOTE : RESOUCRCES SHOULD BE REMOVED AS WE ARE USING THE AUTO MODE

- After updating the config map values you need to delete the previous deployment and create new one.
  another solution found on the github is to create new configmap, add it to the deployment and recreate deployment. last delete the old configmap.

`-----------------------------------------------------------------------------------------
`

IMPROVEMENTS

1> Add the mysql data dynamically. - Done.
2> Check if for deployment we can configure the env variables from deployment it self. - Done.
3> Instead of deployments explore the StatefulSets - Not possible due to complexity

Check if we add Following:

`readinessProble
    	httpGet:
    		path: /api/ready
    		port: 8080`

AND

`livenessProbe
		httpGet:
			path: /api/ready
			port: 8080`
