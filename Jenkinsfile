pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME  = tool 'sonar-scanner'
        DOCKERHUB_USER = 'arpondoc'
        IMAGE_NAME    = "${DOCKERHUB_USER}/bankapp"
        IMAGE_TAG     = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git', url: 'https://github.com/arpon26/mega-project-CI.git'
            }
        }

        stage('Compile') {
            steps { sh 'mvn compile' }
        }

        stage('Testing') {
            steps { sh 'mvn test' }
        }

        stage('Trivy FS Scan') {
            steps { sh 'trivy fs --format table -o fs-report.txt .' }
        }

        stage('Sonar Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=gcbank \
                        -Dsonar.projectName=gcbank \
                        -Dsonar.java.binaries=target
                    '''
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Build') {
            steps { sh 'mvn package -DskipTests' }
        }

        stage('Publish To Nexus') {
            steps {
                configFileProvider([configFile(fileId: 'maven-settings', variable: 'MVN_SETTINGS')]) {
                    sh 'mvn -s $MVN_SETTINGS deploy -DskipTests'
                }
            }
        }

        stage('Docker Image Build & Tag') {
            steps { sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .' }
        }

        stage('Scan Image') {
            steps { sh 'trivy image --format table -o image-report.txt $IMAGE_NAME:$IMAGE_TAG' }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DU', passwordVariable: 'DP')]) {
                    sh '''
                        echo "$DP" | docker login -u "$DU" --password-stdin
                        docker push $IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Update Manifest File in Mega-Project-CD') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'git', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                    sh '''
                        rm -rf Mega-Project-CD
                        git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/arpon26/mega-project-CD.git Mega-Project-CD
                        cd Mega-Project-CD
                        sed -i "s|image: .*bankapp:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|" Manifest/manifest.yaml
                        cat Manifest/manifest.yaml
                        git config user.name "Jenkins"
                        git config user.email "jenkins@local"
                        git add Manifest/manifest.yaml
                        git commit -m "Update image tag to ${IMAGE_TAG}"
                        git push origin main
                    '''
                }
            }
        }
    }
}
