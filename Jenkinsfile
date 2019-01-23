pipeline{
    agent any

    tools {
         maven 'MAVEN-3.3.9'
    }

    parameters{
        string(name: 'BUILD_VERSION', defaultValue: 'LATEST', description: 'Build version to deploy')
        string(name: 'DEPLOY_STAGE', defaultValue: 'DEV', description: 'Build version to deploy')
    }

    environment {
        DOCKER_IMAGE_NAME = "r.tfc.thermofisher.net/edt/platform/device/device-connect-device-file-upload"
        
        MARATHON_APP_ID = "/edt/platform/iot/devicefile/device-connect-device-file-upload"

        MARATHON_DEV2_URL = "http://marathon.dev2.tfc.thermofisher.net"
        MARATHON_TEST_URL = "http://marathon.test.tfc.thermofisher.net"
        MARATHON_STAGE_URL = "http://marathon.stage.tfc.thermofisher.net"
    }

    stages{
        stage('Build'){
            steps {
                dir('DeviceFileUpload'){
                    script {
                        env.POM_VERSION  = readMavenPom().getVersion()                    
                    }
                    sh 'mvn -B clean test cobertura:cobertura'   
                }

                script {
                    echo "Build ${env.POM_VERSION} was successful"
                }
            }
        }

        stage('Release'){
            steps{
                    dir ('DeviceFileUpload'){
                        script{
                            env.GIT_VERSION = env.POM_VERSION.replace("v", "").replace("-SNAPSHOT", "")
                            echo "Releasing version ${env.GIT_VERSION}"                    
                        }
                        sh 'mvn -B clean deploy -DskipITs release:prepare release:perform -Darguments="-DskipITs"'
                    }

                    script {
                        echo "Artifact ${env.GIT_VERSION} was released successfuly to Nexus"

                        echo "Pushing ${env.GIT_VERSION} to registry"

                        sha_value = sh( script: "docker push ${env.DOCKER_IMAGE_NAME}:${env.GIT_VERSION} | grep -Eo \'sha256:[0-9a-z]*\'", returnStdout:true).trim()
                        env.DOCKER_IMAGE_SHA_VALUE = sha_value

                        echo "SHA Value is ${env.DOCKER_IMAGE_SHA_VALUE}"            
                    }
            }
        }


        stage('Deploy-Prepare'){
            steps{
                echo('Preparing Deploy!')
                echo ('This will be used to deploy to specific environments the docker image')
            }
        }

        stage('Deploy-DEV'){
            environment{
                app_env = "dev2"
            }
            
            steps{
                script {
                    echo "Deploying image with SHA value ${env.DOCKER_IMAGE_SHA_VALUE}"

                    marathon credentialsId: 'dev2-marathon-jenkins',
                            url: "${env.MARATHON_DEV2_URL}",
                            id: "${env.MARATHON_APP_ID}",
                            filename: "DeviceFileUpload/DeviceFileUpload-service-server/src/main/marathon/${env.app_env}-marathon.json",
                            docker: "${env.DOCKER_IMAGE_NAME}@${env.DOCKER_IMAGE_SHA_VALUE}",
                            forceUpdate: false,
                            env: [
                                    [name: 'app_env', value: env.app_env],
                                    [name: 'APP_VERSION', value: env.GIT_VERSION]
                            ]

                    echo "Application deployed to Marathon ${env.app_env}!"

                    timeout(time:15, unit: "MINUTES"){
                        input "Deploy to TEST?"
                    }
                }
            }
        }

        stage('Deploy-TEST'){
            environment{
                app_env = "test"
            }
            
            steps{
                script {
                    echo "Deploying image with SHA value ${env.DOCKER_IMAGE_SHA_VALUE}"

                    marathon credentialsId: 'dev2-marathon-jenkins',
                            url: "${env.MARATHON_TEST_URL}",
                            id: "${env.MARATHON_APP_ID}",
                            filename: "DeviceFileUpload/DeviceFileUpload-service-server/src/main/marathon/${env.app_env}-marathon.json",
                            docker: "${env.DOCKER_IMAGE_NAME}@${env.DOCKER_IMAGE_SHA_VALUE}",
                            forceUpdate: false,
                            env: [
                                    [name: 'app_env', value: env.app_env],
                                    [name: 'APP_VERSION', value: env.GIT_VERSION]
                            ]

                    echo "Application deployed to Marathon ${env.app_env}!"

                    sleep time: 2, unit: "MINUTES"

                    echo "Running Integrations Tests against ${env.app_env}"

                    dir('DeviceFileUpload'){
                        sh "mvn -B clean verify -Dskip.surefire.test=true"                    
                    }

                    timeout(time:15, unit: "MINUTES"){
                        input "Deploy to STAGE?"
                    }

                }
            }
        }

        stage('Deploy-STAGE'){
            environment{
                app_env = "stage"
            }
            
            steps{
                script {
                    echo "Deploying image with SHA value ${env.DOCKER_IMAGE_SHA_VALUE}"

                    marathon credentialsId: 'dev2-marathon-jenkins',
                            url: "${env.MARATHON_STAGE_URL}",
                            id: "${env.MARATHON_APP_ID}",
                            filename: "DeviceFileUpload/DeviceFileUpload-service-server/src/main/marathon/${env.app_env}-marathon.json",
                            docker: "${env.DOCKER_IMAGE_NAME}@${env.DOCKER_IMAGE_SHA_VALUE}",
                            forceUpdate: false,
                            env: [
                                    [name: 'app_env', value: env.app_env],
                                    [name: 'APP_VERSION', value: env.GIT_VERSION]
                            ]

                    echo "Application deployed to Marathon ${env.app_env}!"

                    sleep time: 2, unit: "MINUTES"

                    echo "Running Integrations Tests against ${env.app_env}"

                    dir('DeviceFileUpload'){
                        sh "mvn -B clean verify -Dskip.surefire.test=true"                    
                    }
                }
            }
        }
    }
}