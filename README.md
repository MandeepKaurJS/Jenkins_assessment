# Jenkins_assessment

Jenkins : Jenkins is leading open source automation server, Jenkins provides hundreds of plugins to support building, deploying and automating any project.
# What we will do in this assignment
- Automating builds
- Automating Docker image builds
- Automating Docker image upload into AWS ECR
- Automating deploy containers on EKS
# Prerequisites
To follow this tutorial you will need:

1. Jenkins is up and running
2. Docker installed on Jenkins instance.For integrating Docker and Jenkins
3. Steps:
```
Add jenkins user to Docker group
sudo usermod -a -G docker jenkins

Restart Jenkins service
sudo service jenkins restart

Reload system daemon files
sudo systemctl daemon-reload

Restart Docker service as well

sudo service docker stop
sudo service docker start 
 
``` 
4. Docker,Docker pipelines and Kubernetes continous deploy plug-in are installed
5. Repo created in ECR,[Click](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-create.html) here to know how to do that.
6. port 8096 is opened up in firewall rules. 
7. Create an IAM role with AmazonEC2ContainerRegistryFullAccess policy, attach to Jenkins EC2 instance
# How to setup Elastic Container Registry (ECR) for Docker on AWS | How to Create a Repo in ECR for Hosting Docker images | How to Push Docker image into Amazon ECR
- I assigned the role AmazonEC2ContainerRegistryFullAccess to EC2 instance which have installed docker.

- Go to AWS console, click on EC2, select EC2 instance, Go to Actions --> Security--> Modify IAM role.
- Choose the role you have created from the dropdown.
- Select the role and click on Apply.
- Now, I have the access to ECR through EC2 which has docker running on it through AWS CLI which can be installed by

``` sudo apt  install awscli -y```

- Once AWS CLI is installed, you can verify the installation:
```aws --version```

- I logged to AWS ECR using CLI and you will get message with Succeeded value:
``` aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin your_acct_id.dkr.ecr.us-east-2.amazonaws.com```
# EKS Integration with Jenkins
- we need to instaal kubectl in order to communicate with cluster as well as need to create cluster by installing eksctl on our Server and run this Command to create the cluster:
```eksctl create cluster --name demo-eks --region us-east-2 --nodegroup-name my-nodes --node-type t3.small --managed```
- Check if our cluster is created and we can query to cluster using this command
``` kubectl get nodes```
- To integrate EKS with jenkins we need to add credentials to jenkins by 
- Click on Add Credentials, use Kubernetes configuration from drop down.
execute the below command to get kubeconfig info, copy the entire content of the file:
```sudo cat ~/.kube/config```
-This one way to Add kube config information in Jenkins Credentials.
-Jenkins -> Credentials -> Add Credentials -> Select Kind As Kubernates
Configuration ( Kubeconfig) -> Select enter directly radio button -> copy
kubeconfig content from Kubenertes cluster
- AnotherWay by creating .Kube folder inside jenkins server and copy the .kube/config configuration to jenkins and we can directly use kubectl commands in our jenkins server.
- Like That:
  ```   stage('Deploy'){
         /*write steps here to deploy it to EKS cluster with git-sha tag name*/
            when{
                   allOf{
                       expression { params.Git_BRANCH == DEV|STAGING|UAT} 
                       expression { GIT_SHORT_SHA != null}
                }
            steps{
               
                sh "kubectl apply -f deployment.yml"
            }
        }
   
    }```


# Build your pipeline using jenkinsfile
- When everything is setup create your first pipeline by login to jenkins server->new-job->pipeline->git-scm->Save->Apply
# There is some example scripts to defining the stages in jenkins
# I created declarative pipeline in order to acheive my deployment on EKS
- Our pipeline sysntax look like below script and I am defining ENV var to use tham in Pipeline
```
def dockerImage
def GIT_COMMIT =null
def TAG_NAME=null
pipeline {
    agent any
    environment {
        registry = "aws_acnt_ID.dkr.ecr.us-east-1.amazonaws.com/nodeapp"
    }
    //added choice of parameter to deploy the app on specific Branch
     parameters {
        choice(
            name: 'Git_BRANCH',
            choices: ['DEV', 'STAGING', 'UAT', 'PROD'],
            description: 'Passing the Environment')
       }
       Stages{
       stage("any stage"){
       steps{
       //write your required implementation here
       }
       }
       
       }
   }
   
   ```
1. stage first checkout to github repo which contains dockerfile as well as jenkinsfile.

``` 
checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github']]])
```

2. To get the git_commit-sha and git_tag I used this script.
```
script {
         GIT_COMMIT = sh (script: "git log -n 1 --pretty=format:'%h'", returnStdout: true).trim()
          def GIT_SHORT_SHA=GIT_COMMIT.take(8)
          TAG_NAME= sh(script: "git tag --contains | head -1" , returnStdout: true).trim()
          } 
```
3. stage two with steps according to build conditions. 
``` stage('Building image') {
      steps{
        script {
            if( TAG_NAME == null){
                dockerImage = docker.build("$registry"+":${GIT_SHORT_SHA}"
            }else{
                dockerImage = docker.build("$registry"+":${TAG_NAME}"
            }
            
        }
      }
    } 
 ```
4. Push the build images to ECR:

``` 
sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin acont_id.dkr.ecr.us-east-1.amazonaws.com'
 sh "docker push aws_acnt_id.dkr.ecr.us-east-1.amazonaws.com/nodeapp:urtagname" 
```
                
5. Deploy container to EKS using this script in deploy stages with required conditions.

``` 
sh "kubectl apply -f deployment.yml" 
```

# End



