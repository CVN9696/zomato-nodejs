# 🚀 **DevOps Project: ZOMATO Clone App Deployment**

In this **DevOps project**, I demonstrate how to **deploy a ZOMATO Clone App** using a variety of modern DevOps tools and services.

## 🛠️ Tools & Services Used:

1. **GitHub** ![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)
2. **Jenkins** ![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat-square&logo=jenkins&logoColor=white)
3. **SonarQube** ![SonarQube](https://img.shields.io/badge/SonarQube-4E9BCD?style=flat-square&logo=sonarqube&logoColor=white)
4. **Docker** ![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
5. **Kubernetes** ![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
6. **Prometheus** ![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat-square&logo=prometheus&logoColor=white)
7. **Grafana** ![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat-square&logo=grafana&logoColor=white)

---

### Project Stages:

1. **Stage 1** - Deployment of App to Docker Container
2. **Stage 2** - Deployment of App to K8S Cluster with Monitoring

---

### 📂 GitHub Repo Link:  
[**ZOMATO Clone DevOps Project**](#)
https://github.com/rajeshtutta/zomato.git
---

## Happy learning!  
<img src="https://media.licdn.com/dms/image/v2/D5603AQHJB_lF1d9OSw/profile-displayphoto-shrink_800_800/profile-displayphoto-shrink_800_800/0/1718971147172?e=1735776000&v=beta&t=HC_e0eOufPvf8XQ0P7iI9GDm9hBSIh5FwQaGsL_8ivo" alt="Kastro Profile Image" width="100" height="100" style="border-radius:50%;">

KASTRO KIRAN V




_++++++++++++++++++++++ satyanaratyana

pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node23'
 }

    environment {
        SONARQUBE_ENV = 'SonarQube'
        DOCKER_IMAGE = "vamsichamarthi/zomato"
        AWS_DEFAULT_REGION = 'us-west-1'
        RECIPIENTS = 'vamsinath.05@gmail.com'
    }

    stages {

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/CVN9696/zomato-nodejs.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build Application') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'npm test || true'
            }
        }

        stage('Package Artifact') {
            steps {
                sh 'zip -r zomato-build.zip build/'
            }
        }

        stage('SonarQube Analysis') {
    steps {
        script {
            def scannerHome = tool 'SonarScanner'

            withSonarQubeEnv('SonarQube') {
                sh """
                ${scannerHome}/bin/sonar-scanner \
                -Dsonar.projectKey=zomato \
                -Dsonar.projectName=Zomato-App \
                -Dsonar.sources=src \
                -Dsonar.projectVersion=${BUILD_NUMBER}
                """
            }
        }
    }
}

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-cred',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                    curl -v -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file zomato-build.zip \
                    http://localhost:8081/repository/raw-hosted/zomato-build-${BUILD_NUMBER}.zip
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t $DOCKER_IMAGE:${BUILD_NUMBER} .
                docker tag $DOCKER_IMAGE:${BUILD_NUMBER} $DOCKER_IMAGE:latest
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $DOCKER_IMAGE:${BUILD_NUMBER}
                    docker push $DOCKER_IMAGE:latest
                    docker logout
                    '''
                }
            }
        }

//         stage('Install Helm') {
//              steps {
//                  sh '''
//                curl -LO https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
//                tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
//                mv linux-amd64/helm helm
//                chmod +x helm
//                '''
//             }
//         }
        
//         stage('Verify Helm') {
//     steps {
//         sh './helm version'
//     }
// }

// stage('Add Helm Repo') {
//     steps {
//         sh './helm repo add prometheus-community https://prometheus-community.github.io/helm-charts'
//         sh './helm repo update'
//     }
// }

// stage('Install Monitoring Stack') {
//     steps {
//         // sh './helm install monitoring prometheus-community/kube-prometheus-stack'
//         sh './helm upgrade --install monitoring prometheus-community/kube-prometheus-stack'
//     }
// }

        stage('Configure EKS Access') {
    steps {
        sh '''
        export PATH=$PATH:/usr/local/bin
        aws eks update-kubeconfig --region us-west-1 --name myclusterr
        /usr/local/bin/kubectl config current-context
        '''
    }
}

        stage('Deploy to EKS') {
            steps {
                sh '''
                
                kubectl apply -f deployment.yml
                kubectl apply -f service.yml
                '''
            }
        }
        // stage('Deploy to EKS') {
        //     steps {
        //         {
        //             sh '''
        //             aws eks update-kubeconfig --region us-west-1 --name myclusterr
        // aws eks --region $AWS_REGION update-kubeconfig --name $EKS_CLUSTER
        //             kubectl apply -f deployment.yml
        //             kubectl apply -f service.yml
        //             '''
        //         }
        //     }
        // }
    }

    post {
        success {
            emailext(
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build succeeded!\n\nCheck: ${env.BUILD_URL}",
                to: "${RECIPIENTS}"
            )
        }

        failure {
            emailext(
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build failed!\n\nCheck: ${env.BUILD_URL}",
                to: "${RECIPIENTS}"
            )
        }

        always {
            archiveArtifacts artifacts: 'zomato-build.zip', fingerprint: true
        }
    }
}
