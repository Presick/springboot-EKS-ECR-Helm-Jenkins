## PROJECT ##

##### Context
We want to deploy a Springboot application on an EKS cluster using Helm, Jenkins as CI tool, ECR as Container image registry. 

##### Tools
+ AWS
+ Jenkins CE
+ SCM (git/github)
+ Maven plugin
+ Helm
+ ECR
+ EKS cluster

##### Infrastructure
+ Cloud provider account: AWS
+ 1 EC2 instance t2.medium OS ubuntu 24-04
+ ECR repository
+ 2 Volumes
+ amazon ELB
+ 1 Autoscaling group
+ Create security group and open port 22, 8080, which are the port we will be using for this project

![image](https://github.com/user-attachments/assets/ceb020b3-6ba3-41dd-b8f7-36e347eac2df)

![image](https://github.com/user-attachments/assets/4d617131-39ff-4506-8377-31dc0db4c50c)

![image](https://github.com/user-attachments/assets/33bd5145-d66c-4363-b480-1e638023943f) 

![image](https://github.com/user-attachments/assets/48ddcdfc-857c-4293-baf4-4474fd926d5f)


##### WORKFLOW

##### Description

+ Once the developer commit the code and once this one is merged to the main branch, a build is triggered on Jenkins
+ The artefact is successively built( Maven plugin), then the container image is built and uploaded into ECR registry.
+ The container image is then pull and deployed as part of a helm chart.

+ ![image](https://github.com/user-attachments/assets/a197465d-d8b2-4014-a013-17b52d9a5a12)

 ##### step 1
On Jenkins dashboard: + NewItem - Name the project (myHelmDeployment) - Select Pipeline - click OK 
![image](https://github.com/user-attachments/assets/74c1cf31-1ba6-473f-b290-5450290216e2) 

##### step 2 Setup Maven
![image](https://github.com/user-attachments/assets/d48e9abc-d29d-4327-95b0-96b63d226701)

##### step 3 set Helm chart
+ Add Docker image details to download from ECR before deploying to EKS cluster
open mychart/values.yaml.
![image](https://github.com/user-attachments/assets/e220bbb5-64e0-48d8-a88e-eebce61205ba)

+ Enter service type as LoadBalancer
And also
open mychart/templates/deployment.yaml and change containerPort to 8080

![image](https://github.com/user-attachments/assets/12647652-d72a-4898-876d-ed18bf101beb)

##### step 4 Create EKS cluster


eksctl create cluster --name my-demo-eks --region us-east-1 --nodegroup-name my-demo-nodes --node-type t3.small --managed --nodes 2

###### confirm that EKS cluster is up and running

eksctl get cluster --name my-demo-eks --region us-east-1

###### Update Kube config

aws eks update-kubeconfig --name demo-eks --region us-east-1
##### step 5 create namespace in EKS

kubectl create ns helm-deployment

##### step 6 Jenkins - Pipeline

pipeline {
    agent any
    environment{
        registry = "584196443731.dkr.ecr.us-east-1.amazonaws.com/my-docker-repo"
        
    }
    tools { 
      maven 'maven3.6.3' 
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scmGit(branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/Presick/docker-spring-boot/']])
            }
        }
        
        stage ("Build Jar"){
            steps {
                sh "mvn clean install"
            }
        }
        
        stage ("Building Docker Image") { 
            steps{
                script {
                    docker.build registry + ":$BUILD_NUMBER"
                }  
            }
        }
        
        stage ("Push Image to ECR") {
            steps {
                script {
                    sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 584196443731.dkr.ecr.us-east-1.amazonaws.com"
                    sh "docker push 584196443731.dkr.ecr.us-east-1.amazonaws.com/my-docker-repo:$BUILD_NUMBER"
                }
            }
        }
        
        stage ("Helm Deploy") {
            steps {
                script {
                    sh "helm upgrade first --install mychart --namespace helm-deployment --set image.tag=$BUILD_NUMBER"
                }
            }
        }
    }
}

##### step 7 Deployment

![image](https://github.com/user-attachments/assets/94d59a04-03e1-4338-aae9-132c619863bb)


## Source code:
https://github.com/Presick/docker-spring-boot
