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
                withCredentials([usernamePassword(credentialsId: 'jfrog-dev', passwordVariable: 'jfrog-pass', usernameVariable: 'jfrog-user')]) {
                sh '''cp webapp/target/webapp.war webapp/target/webapp_$BUILD_ID.war
                echo "Username: ${jfrog-user}"
                echo "Password: ${jfrog-pass}"
                def jfrogCreds = credentials('jfrog-dev')
                def jfrog-user = jfrogCreds.username
                def jfrog-pass = jfrogCreds.password
                curl -u"${jfrog-user}:${jfrog-pass}" -T webapp/target/webapp_$BUILD_ID.war "http://20.185.219.50:8081/artifactory/jfrog-dev/jk-2_08.07.2023/"'''
                }
            }
        }
        
        stage('Docker Build & Push image'){
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred', passwordVariable: 'dockerhubpassword', usernameVariable: 'dockerhubuser')])  {
                    sh "docker login -u $dockerhubuser -p $dockerhubpassword"
                }
                sh '''docker build -t sample-1:v1.$BUILD_ID .
                docker tag sample-1:v1.$BUILD_ID jyothibasuk/sample-1:v1.$BUILD_ID
                docker push jyothibasuk/sample-1:v1.$BUILD_ID
                docker rmi sample-1:v1.$BUILD_ID
                docker rmi jyothibasuk/sample-1:v1.$BUILD_ID''' 
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
