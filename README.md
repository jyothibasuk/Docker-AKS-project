CICD Pipeline for Deploying JAVA application into Azure Kubernetes Service cluster from Azure container registry

Step-1
Create Linux machine i VM    .

Here size of vm is D2s_V3(8GB RAM & 2 Vcpus) – high cost, you can use Standard_B2(4GB RAM & 2 Vcpus) to reduce cost -choose as per your load requirement.
 
Here you can choose SSH key or password

You need VNET, it can be created here directly.

Always assign your subnet to NSG for better security & traffic control – it can be created here directly.

Click on create

Click on ip address and make it static & save (it will add cost)

Finally set the NSG rules, before accessing VM
Here, since it is PoC it is opened to everyone. In real time case scenarios never keep the incoming source  open for public(any) which is connecting to your target VM/server.

Now login to server (I have used putty)
 

 
Step-2
Install GIT, MAVEN, JENKINS, SONARQUBE, JFROG, DOCKER in the above VM.
After connecting and logging into the VM, 
Installing GIT, MAVEN, JAVA & set the path
apt-get update -y
apt-get install git maven -y
apt-get install openjdk-8*
mvn -version
java --version
JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export JAVA_HOME
PATH=$PATH:$JAVA_HOME
MVN_HOME=/usr/share/maven
export MVN_HOME
PATH=$PATH:$MVN_HOME

Installing JENKINS 
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install Jenkins
post installation of Jenkins
1)	Login with the <publicip>:8080
2)	Provide password
3)	Go with default plugins
4)	Set the path for JAVA & MAVEN
5)	Also add any plugins required




Installing DOCKER 
To install docker there are two steps, so I am following the quicker one
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -G docker ubuntu
sudo usermod -G docker Jenkins
sudo systemctl restart Jenkins
also exit the putty & reconnect for the ubuntu user to access docker.
docker info (run as normal user - below output should be; note: don’t run as sudo or root user)
 


Installing SONARQUBE 
We are not directly installing sonarqube in the VM. Rather we are using docker container to run SonarQube. 
docker run -d -p 9000:9000 -p 9092:9092 --name sonarqube sonarqube
 
Login with the <publicip>:9000 and create project.
 
Installing JFROG 
sudo apt update
wget -qO - https://api.bintray.com/orgs/jfrog/keys/gpg/public.key | sudo apt-key add -
echo "deb https://jfrog.bintray.com/artifactory-debs bionic main" | sudo tee /etc/apt/sources.list.d/jfrog.list
sudo apt update
sudo apt install jfrog-artifactory-oss
sudo systemctl start artifactory.service
sudo systemctl enable artifactory.service
systemctl status artifactory.service
 
Login with the <publicip>:8082 and create artifact repository.
 
GIT-HUB
Find and Clone sample JAVA application (remember – it should have POM.XML file)

Step -3
CI – Pipeline
The major stages are
1)	Checkout code from github
2)	Code Analysis
3)	Building the code
4)	Pushing the artifacts to jrog

Write the Jenkins groovy script accordingly.
pipeline {

    agent any

    stages {

        stage ('checkout') {

            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/xyz/xyz.git']]])
            }
        }

        stage ('Code Analysis') {

            steps {
                sh '''mvn sonar:sonar \\
                -Dsonar.projectKey=Poc-AKS1 \\
                -Dsonar.host.url=http://52.140.116.20:9000 \\
                -Dsonar.login=e5ae4daa1e4c7cffe91ed213468a069ce581ff69'''
            }
        }

        stage ('Build') {

            steps {

                sh 'mvn clean package'
                
            }
        }
        stage ('Push Artifacts to Jfrog') {

            steps {
                sh '''cp webapp/target/webapp.war webapp/target/webapp_$BUILD_ID.war
                curl -uadmin:AP34mCp3r3nLNeLoHTaGnbrAuEJ -T webapp/target/webapp_$BUILD_ID.war "http://52.140.116.20:8081/artifactory/example-repo-local/"'''
                
            }
        }
}
}

Step-4
Create azure container repository
 
After that click on create directly.
Finally it looks as below.
 

Now use the below details and create azure container registry credentials in Jenkins, this is used to push images.
 


Step-5
Create AKS cluster in azure using azure-cli
az aks create --resource-group JK-TCS --name AKSCluster --node-count 2 --enable-addons monitoring --generate-ssh-keys
az aks get-credentials --resource-group JK-TCS --name AKSCluster --overwrite-existing
to view the credentials
cat ~/.kube/config
Now copy the output and save in Jenkins as aks credentials.


Refer below documentation and video for 
1)	creating service principal & assign to your azure container registry.
https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-service-principal
https://www.youtube.com/watch?v=3Hu_PHfAaio&list=PLKEhvrzO28rYrdwVmf0N2u8_AKfBAA5hQ&index=3


example:
Log in to Docker with service principal credentials
docker login aksacrtcspoc.azurecr.io --username 81e4269a-150e-48b6-8341-56526482f74c --password JR67.SPGt-TvdEKlGxABVMqtibYsu8uz6O
az acr login --name aksacrtcspoc
To add secret into the AKS cluster
kubectl create secret docker-registry aksacrtcspoc --namespace default --docker-server=aksacrtcspoc.azurecr.io --docker-username=81e4269a-150e-48b6-8341-56526482f74c --docker-password=JR67.SPGt-TvdEKlGxABVMqtibYsu8uz6O


Step-6
Create Dockerfile
Creating build and push to azure container registry
FROM tomcat:latest
COPY webapp/target/webapp.war /usr/local/tomcat/webapps


Creating deployment & service yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aks-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: jyothibasuk/poc-1:v1.$BUILD_ID
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
# service type loadbalancer
---
apiVersion: v1
kind: Service
metadata:
  name: aks-svc
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30008
  type: LoadBalancer




Step-7
Final CI – CD Pipeline
Jenkins groovy script
pipeline {

    agent any

    environment {
        registry = "aksacrtcspoc.azurecr.io"
    }

    stages {

        stage ('checkout') {

            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/xyz/xyz.git']]])
            }
        }

        stage ('Code Analysis') {

            steps {
                sh '''mvn sonar:sonar \\
                -Dsonar.projectKey=Poc-AKS1 \\
                -Dsonar.host.url=http://52.140.116.20:9000 \\
                -Dsonar.login=e5ae4daa1e4c7cffe91ed213468a069ce581ff69'''
            }
        }

        stage ('Build') {

            steps {

                sh 'mvn clean package'
                
            }
        }

        stage ('Push Artifacts to Jfrog') {

            steps {
                sh '''cp webapp/target/webapp.war webapp/target/webapp_$BUILD_ID.war
                curl -uadmin:AP34mCp3r3nLNeLoHTaGnbrAuEJ -T webapp/target/webapp_$BUILD_ID.war "http://52.140.116.20:8081/artifactory/example-repo-local/"'''
                
            }
        }
        
        stage('ACR-Build & Push image'){
            steps {
                withCredentials([usernamePassword(credentialsId: 'acr_cred', passwordVariable: 'acrpswd', usernameVariable: 'aksacrtcspoc')]) {
                    sh "docker login aksacrtcspoc.azurecr.io -u $aksacrtcspoc -p $acrpswd"
                }
                sh '''docker build -t poc-1:v1.$BUILD_ID .
                docker tag poc-1:v1.$BUILD_ID $registry/poc-1:v1.$BUILD_ID
                docker push $registry/poc-1:v1.$BUILD_ID
                docker rmi poc-1:v1.$BUILD_ID
                docker rmi $registry/poc-1:v1.$BUILD_ID'''
            }
        }

        stage ('Deploy to AKS Cluster') {

            steps {
                script {
                    
                    kubernetesDeploy(
                    configs: 'acr-deployment.yaml',
                    kubeconfigId: 'k8s-cluster',
                    ) 
                }
                
            }
        }

    }
}


------------------------------
Jenkins Plugins Installed
Kubernetes
Kubernetes Continuous Deploy 1.0.0
Docker Pipeline
Docker

