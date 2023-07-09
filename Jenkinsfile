pipeline {

    agent any

    stages {

        stage ('checkout') {

            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/jyothibasu/hello-world.git']]])
            }
        }

        stage ('Code Analysis') {

            steps {
                sh '''mvn clean verify sonar:sonar \
                -Dsonar.projectKey=jk-2_08.07.2023 \
                -Dsonar.projectName='jk-2_08.07.2023' \
                -Dsonar.host.url=http://20.185.219.50:9000 \
                -Dsonar.token=sqp_ad648a6a10b22df7487509fee76e0172305e2ba8'''
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
                curl -uadmin:cmVmdGtuOjAxOjE3MjA0MTg2MTE6Q1ZTYk00STZVQTZxbmY5Z2x6N1kzWVdETXhz -T webapp/target/webapp_$BUILD_ID.war "http://20.185.219.50:8081/artifactory/jfrog-dev/jk-2_08.07.2023/"'''
                
            }
        }
        
        stage('Docker Build & Push image'){
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker_cred', passwordVariable: 'dockerhubpassword', usernameVariable: 'dockerhubuser')]) {
                    sh "docker login -u $dockerhubuser -p $dockerhubpassword"
                }
                sh '''docker build -t poc-1:v1.$BUILD_ID .
                docker tag poc-1:v1.$BUILD_ID jyothibasuk/poc-1:v1.$BUILD_ID
                docker push jyothibasuk/sample-1:v1.$BUILD_ID
                docker rmi poc-1:v1.$BUILD_ID
                docker rmi jyothibasuk/poc-1:v1.$BUILD_ID''' 
            }
        }

        stage ('Deploy to AKS Cluster') {

            steps {
                script {
                    
                    kubernetesDeploy(
                    configs: 'aks-deployment.yaml',
                    kubeconfigId: 'k8s-cluster',
                    ) 
                }
                
            }
        }

    }
}
