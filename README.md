# DevSecOps-project-deployment-process
end to end DevSecOps project deployment process

## Deploy a project of end-to-end with DevSecOps 

1. Log in to the AWS account & take an instance of t2.large 
2. Connect it.
3. Update the instance 
```bash
sudo apt update -y  
```
## A. Setup of Tools
## 1. setup of Docker and Docker-Compose
1. we add security in this project, so we need to install docker 
```bash
sudo apt install docker.io -y  
docker -v
docker ps 
```
2. need to give permission 
```bash
sudo usermod -aG docker $USER && newgrp docker 
docker ps   
```
3. now install docker-compose v2
```bash
sudo apt install docker-compose-v2 -y 
docker compose version 
```
## 2. Jenkins's setup
1. Now jenkins install. need Java to run Jenkins, then first install Java and then Install Jenkins's LTS version below link.
2. [Jenkins installation](https://www.jenkins.io/doc/book/installing/linux/)
** by default jenkins runs in port no.:8080, need to open inbound rule from security group of 8080
3. status check of jenkins
```bash 
sudo systemctl status jenkins
```
4. access jenkins in the browser by:
    ** instance_public_ip:8080
5. need to unlock jenkins then, jenkins give the path of admin password in the window, copy that path and paste it in instance cli 
```bash
sudo cat path_from_jenkins_unlock_window  
``` 
1. install suggested plugin
2. create first admin user and then log in 
3. add jenkins to docker group to run docker command in jenkins pipeline 
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart docker
sudo systemctl restart jenkins 
```
## 3. SonarQube's setup
1. now install sonarqube by docker image and default port no. of sonarqube is :9000 (Security scan of code, check vulnerabilities in code, continuous inspection of code quality)
```bash
docker run -itd --name sonarqube-server -p 9000:9000 sonarqube:lts-community  
```
2. check conatiner is running or not | there is sonarqube is running in :9000 | open 9000 port no. from security group 
```bash
docker ps   
``` 
3. access sonar qube in the browser by 
    ** public_ip_of_instance:9000
4. log in | name: admin , pass: admin | add new credentials

## 4. Trivy's  setup
1. trivy install from it's official website 
2. [trivy installation](https://trivy.dev/latest/getting-started/installation/)
** designed to detect vulnerabilities in various software artifacts and deployments.

## 5. OWASP setup
1. plugins insatll: manage jenkins --> plugins --> available plugin --> search
2. plugins install in the jenkins:
    1. SonarQube Scanner,
    2. Sonar Quality Gates,
    3. OWASP dependency,
    4. Docker,
    5. Pipeline view,

## 6. Integration of Jenkins and SonarQube by URL / Webhook 
## i. In SonarQube --
1. in SonarQube go to:
    Administration --> configuration --> webhooks --> create --> name: Jenkins , URL: http://jenkins_url_paste_here/sonarqube-webhook/ --> then create 
2. create token:
    security --> users --> create token tap on 3 dot --> name: jenkins-sonar, 30 days --> then generate --> copy token and save it  
## ii. In Jenkins now 
1. In jenkins add credentials go to:
    Manage jenkins --> credentials --> global --> add credentials --> select - kind: secret text, paste token of sonar, id: sonar,description: sonar
2. now add sonarqube insatllation in jenkins go to:
    manage jenkins --> system --> sonarqube server --> add sonar --> name: sonar, server url: http://sonarqube_server_url_link_paste_here:9000 , add authentication token --> save it 

## iii. add sonaqube quality gate (sonar scanner tool)
1. add sonarqube quality gate in jenkins goto:
    manage jenkins --> tools --> sonarqube scanner installations --> add --> name: sonar , select automatically and add version --> save it 

** this is completly integrate sonarqube and jenkins with each other 

## 7. setup of OWASP in jenkins (Tool | dependency check tool)
1. OWASP tool setup in jenkins go to:
    manage jenkins --> tool --> dependency-check installations --> add --> name: dc , select install automatically & add installer - select github.com - take version --> save it  

## B. Now create pipeline in jenkins
## 1. create declarative pipeline here
1. in jenkins's dashboard go to :
    dashboard --> new item --> name: project_name, select - pipeline --> ok 
2. there is form for pipeline:
    1. description: This is CI/CD pipeline for project 
    2. select - github project --> paste here project url from github
    3. select - throttle buid
    4. for automatic trigger polling - select --> Github hook trigger for GITScm polling
    5. now write pipeline with grovvy syntax:
    
    ```bash
    pipeline{
        agent any 

        environment{
            SONAR_HOME= tool "sonar" 
            IMAGE_NAME= "react-app-omniscient-frontend"
            COMPOSE_FILE= "docker-compose.yml"
        }
    
        stages{
            stage("1.Code clone from Github"){
                steps{ 
                    echo "code clone from github"
                    git url: project_url_paste_here_from_github , branch= "main"
                }
            }
            stage("2.SonarQube Quality Analysis"){
                steps{
                    echo "SonarQube Quality Analysis"
                    withSonarQubeEnv("sonar"){
                        sh "$SONAR_HOME/bin/sonar-scanner -Dsonar.projectName=project_name -Dsonar.projectKey=project_name"
                    }
                }
            }
            stage("3.OWASP Dependency Check"){
                steps{
                    dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'dc'
                    dependencyCheckPublisher pattern: "**/dependeny-check-report.xml"
                }
            }
            stage("4.SonarQuality Gate Scan"){
                steps{
                    timeout(time: 2, unit: "MINUTES"){
                        waitForQualityGate abortPipeline: false
                    }
                }
            }
            stage("5.Trivy File system scan"){
                steps{
                    sh "trivy fs --format table -o trivy-fs-report.html ."
                }
            }
            stage("6.Docker Build an image"){
            steps {  
                echo "Building the image of project"
                script {                    
                    def imageTag = "adityabahuguna2002/${IMAGE_NAME}:${BUILD_NUMBER}"
                    sh "docker build -t ${imageTag} ."
                    }
                }
            }
            stage("7.Docker Push to Push an Image to Docker Hub"){
            steps {  
                echo "Docker push to push an image to docker hub" 
                script {                    
                    def imageTag = "adityabahuguna2002/${IMAGE_NAME}:${BUILD_NUMBER}"                    
                        withDockerRegistry(credentialsId: 'docker-hub', toolName: 'docker'){   
                            sh "docker push ${imageTag}"
                        }
                    }
                }
            }
            stage("7.Deploy the project using Docker Compose"){
                steps{
                    echo "Docker compose Deploy the project"
                    script{
                        sh """
                            docker compose down || true
                            docker compose up -d --build
                        """
                    }
                }
            }
        }

        post{
            success{
                echo "Deployed Successfully"
            }
            failure{
                echo "Deployment Failed"
            }
        }
    }
    ``` 




