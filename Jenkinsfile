pipeline {

    agent any
/*
	tools {
        maven "maven3"
    }
*/
    environment {
        registry = "anhnm/vproapp" //dockerhub account/image. Can use any name with image
        registryCredential = "dockerhub" // use credentials id created on Jenkins
    }

    stages{
        stage('BUILD'){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {

            environment {
                scannerHome = tool 'mysonarscanner4'
            }

            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App Image'){
            steps {
                script {
                    //Run Docker build command
                    //Dockerfile should be in same dir in source code
                    dockerImage = docker.build registry + ":V$BUILD_NUMBER" // use registry var defined in environment
                }
            }
        }

        stage('Upload Image'){
                    steps {
                        script {
                            docker.withRegistry('', registryCredential) { //pass registryCredential defined in environment to this function
                                dockerImage.push("V$BUILD_NUMBER") //dockerImage var defined at line 79
                                dockerImage.push("latest") //also push with tag latest
                            }
                        }
                    }
                }

        stage('Remove Unused Docker Images'){ //remove the images on Jenkins after pushed to dockerhub, as everytime run the job it create new image
                    steps {
                        sh "docker rmi $registry:V$BUILD_NUMBER" //registry is var in environment
                    }
                }

        stage('Kubernetes Deploy'){
        agent {label 'KOPS'} //run form agent with label 'KOPS'
                            steps {
                                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
                            }
                        }
    }


}