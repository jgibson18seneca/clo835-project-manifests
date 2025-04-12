# clo835-project-manifests

This branch is for the YAML mainfests used in the project
Provision Order:
Secrets
ConfigMaps
Persistent Volumes
Presistent Volume Claims
Deployments
Services
# Command String
## Test Docker Images Locally
### Login to ECR 
```
export ECR=[youracounturl]/project-ecr
```
```
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin [youraccounturl]
```
### Build Images
```
docker build -t $ECR:db-v1 -f Dockerfile_mysql .
```
```
docker build -t $ECR:app-v1 -f Dockerfile .
```
### Export Variables
```
export DBPORT=3306 && export DBHOST=172.17.0.2 && export DBUSER=root && export DBPWD=something && export DATABASE=employees && export DBNAME=employees
```
### Run Containers
```
docker run -d --name db -e MYSQL_ROOT_PASSWORD=something $ECR:db-v1
```
```
docker run -d --name app -p 81:81 -e DBHOST=$DBHOST -e DBPORT=$DBPORT -e DBUSER=$DBUSER -e DBPWD=$DBPWD $ECR:app-v1
```
```
docker logs app
```
### Check connectivity
```
curl localhost:81
```
### Push Images to Repo (Optional for testing)
```
docker push $ECR:db-v1
```
```
docker push $ECR:app-v1
```
### Remove Images and Containers
```
docker rm -vf $(docker ps -aq) && docker rmi -f $(docker images -aq)
```
## Setup Deployment in AWS EKS
### Clone or pull from Manifests Repo
#### Clone (First time)
```
git clone https://github.com/jgibson18seneca/clo835-project-manifests
```
Username and Password (GitHub token) may be needed for first time clone

### Copy manifests to specific folder
```
cp configmaps/* deployments/* eks/* pv/* pvc/* secrets/* services/* ../manifests
```
### Use credentials from AWS Academy AWS Details and copy them into ~/.aws/credentials file
```
vim ~/.aws/credentials
```
### install jq
```
sudo yum -y install jq gettext bash-completion moreutils
```
### Install eksctl
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```
```
sudo mv -v /tmp/eksctl /usr/local/bin
```
## Initial Steps for Kubernetes
### Create EKS Cluster
```
eksctl create cluster -f eks_config.yaml
```
### Update kubeconfig
```
aws eks update-kubeconfig --name clo835 --region us-east-1
```
## Deploy Kubernetes Application
### Alias
```
alias k=kubectl
```
### Create namespace
```
k create ns final
```
### Apply and verify ConfigMap
```
k apply -f configmap.yaml -n final
```
```
k get configmap proj-configmap -n final
```
### Apply Secret
```
k apply -f app-secret.yaml -n final
```
### Apply PV
#### Create EBS Volume (Can be done before deployment of EKS)
```
aws ec2 create-volume --volume-type gp2 --size 10 --availability-zone us-east-1a
```
Get volume ID and use in Deployment

#### Ensure PV/PVC Driver is installed
```
eksctl create addon --name aws-ebs-csi-driver --cluster clo835 --service-account-role-arn arn:aws:iam::134986800938:role/LabRole --force
```
#### Create PV
```
k apply -f mysql-pv.yaml -n final
```
```
k get pv 
```
### Apply PVC
```
k apply -f mysql-pvc.yaml -n final
```
```
k get pvc -n final 
```
### Create Database Deployment and Service
```
k apply -f db-dp.yaml -n final
```
```
k apply -f db-svc.yaml -n final
```
### Create App Deployment and Service
```
k apply -f app-dp.yaml -n final
```
```
k apply -f app-svc.yaml -n final
```
```
k port-forward svc/app-service 8080:8080 -n final
```
### Check Connectivity
```
curl localhost:8080
```

Change S3 Bucket Image in configmap.yaml

### Re-apply configmap
```
k apply -f configmap.yaml -n final
```
### Restart App Deployment
```
k rollout restart deployment app-dp -n final
```
```
k port-forward svc/app-service 8080:8080 -n final 
```
### Restart DB Deployment
```
k rollout restart deployment db-dp -n final
```
### Other Kubernetes Commands
```
k get nodes
k get all -n final
k get pv -n final
k get pvc -n final
k cluster-info --context kind-kind
```
### Remove all resources
```
k delete deployments --all -n final
```
```
k delete services --all -n final
```
```
k delete pvc mysql-pvc -n final
``` 

### Remove namespaces
```
k delete namespace final
```
### Delete Cluster
```
eksctl delete cluster -f eks_config.yaml
```
### Remove all images
```
docker rmi -f $(docker images -aq)
```
