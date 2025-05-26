pipeline {
    agent any
    
    tools {
        maven "M3"
        jdk "JDK21"
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerCredential')
        REGION = "ap-northeast-2"
        AWS_CREDENTIALS_NAME = "AWSCredentials"
    }
    
    stages {
        stage('Git Clone') {
            steps {
                echo 'Git Clone'
                git url: 'https://github.com/CHOI-SB/spring-petclinic.git',
                    branch: 'main'
            }
            post {
                success {
                    echo 'Git Clone Success'
                }
                failure {
                    echo 'Git Clone Fail'
                }
            }
        }
        
        // Maven build 작업
        stage('Maven Build') {
            steps {
                echo 'Maven Build'
                sh 'mvn -Dmaven.test.failure.ignore=true clean package' // Test error 무시
            }
        }
        
        // docker image 생성
        stage('Docker Image Build') {
            steps {
                echo 'Docker Image Build'
                dir("${env.WORKSPACE}") {
                    sh '''
                        docker build -t spring-petclinic:$BUILD_NUMBER .
                        docker tag spring-petclinic:$BUILD_NUMBER sobin0401/spring-petclinic:latest
                        '''
                }
            }
            
        }

        // docker image push
        stage('Docker Image Push') {
            steps {
                sh '''
                    echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                    docker push sobin0401/spring-petclinic:latest
                '''
            }
        }

        // remove docker image
        stage('Remove Docker Image') {
            steps {
                sh '''
                docker rmi spring-petclinic:$BUILD_NUMBER
                docker rmi sobin0401/spring-petclinic:latest
                
                '''
            }
        }
        stage('Upload S3') {
        steps {
            echo "Upload to S3"
            dir("${env.WORKSPACE}") {
                sh 'zip -r scripts.zip ./scripts appspec.yml'
                withAWS(region:"${REGION}",credentials:"${AWS_CREDENTIALS_NAME}"){
                    s3Upload(file:"scripts.zip", bucket:"team1-codedeploy-bucket")
                    }
                    sh 'rm -rf ./scripts.zip' 
                }
            }    
        }
        stage('Codedeploy Workload') {
            steps {
               echo "create Codedeploy group"   
                sh '''
                    aws deploy create-deployment-group \
                    --application-name user00-code-deploy \
                    --auto-scaling-groups user00-asg \
                    --deployment-group-name user00-code-deploy-${BUILD_NUMBER} \
                    --deployment-config-name CodeDeployDefault.OneAtATime \
                    --service-role-arn arn:aws:iam::257307634175:role/user00-codedeploy-service-role
                    '''
                echo "Codedeploy Workload"   
                sh '''
                    aws deploy create-deployment --application-name user00-code-deploy \
                    --deployment-config-name CodeDeployDefault.OneAtATime \
                    --deployment-group-name user00-code-deploy-${BUILD_NUMBER} \
                    --s3-location bucket=user00-codedeploy-bucket,bundleType=zip,key=deploy.zip
                    '''
                    sleep(10) // sleep 10s
            }

        // stage('SSH Publish'){
        //     steps {
        //         echo 'SSH Publish'
        //         sshPublisher(publishers: [sshPublisherDesc(configName: 'target', 
        //         transfers: [sshTransfer(cleanRemote: false, 
        //         excludes: '', 
        //         execCommand: '''
        //         docker rm -f $(docker ps -aq)
        //         docker rmi $(docker images -q)
        //         docker run -d -p 8080:8080 --name spring-petclinic sobin0401/spring-petclinic:latest
        //         ''', 
        //         execTimeout: 120000, flatten: false, 
        //         makeEmptyDirs: false, 
        //         noDefaultExcludes: false, 
        //         patternSeparator: '[, ]+', 
        //         remoteDirectory: '', 
        //         remoteDirectorySDF: false, 
        //         removePrefix: 'target', 
        //         sourceFiles: 'target/*.jar')], 
        //         usePromotionTimestamp: false, 
        //         useWorkspaceInPromotion: false, 
        //         verbose: false)])
        //     }
        // }
    }
}
