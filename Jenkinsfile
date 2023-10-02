pipeline {
agent any
tools {nodejs "nodejs"}
stages {
        // stage('user-identification') {
        //  steps {
        //      script {
        //             try {
        //                 withCredentials(
        //                     [string(credentialsId: 'user_verification', variable: 'user_verification')]
        //                 ) 
        //                 {
        //                     def askpass = input(
        //                     message: 'Please enter the password',
        //                     parameters: [
        //                         password(defaultValue: '',
        //                                 description: '',
        //                                 name: 'password')])  
        //                     if ("$user_verification" == "$askpass") {                                                           
        //                         sh """ 
        //                         echo "Password Matched"
        //                         """
        //                     }
        //                     else {
        //                         sh "echo Wrong Password"
        //                         sh "exit 1"
        //                     }
        //                 }
        //             }
        //             catch (Exception e) {
        //                 echo "FAILED ${e}"
        //                 currentBuild.result = 'FAILURE'
        //                 throw e
        //             }
        //         }
        //     }
        // }

    stage('Remove old code') {
        steps {
            script {
            try{  
                sh "rm -rf *"
             }
    catch(Exception e) {
          echo "FAILED ${e}"
          currentBuild.result = 'FAILURE'
          throw e
        }             
            }
        } }
 stage("Code Checkout from GitHub") {
  steps {
      script {
               try{        
          sh "rm -rf *"
          git branch: "main" , credentialsId: 'creds' ,url: 'https://github.com/Bharat2912/scrum-task.git'
   }
        catch(Exception e) {
          echo "FAILED ${e}"
          currentBuild.result = 'FAILURE'
          throw e
        } }
  }
 }

//   stage('Code Quality Check via SonarQube') {
//   steps {
//       script {
//       try {
//       def scannerHome = tool 'sonarqube';
//           withSonarQubeEnv("sonarqube") {
//           sh "${tool("sonarqube")}/bin/sonar-scanner \
//       -Dsonar.projectKey=scrum-task \
//       -Dsonar.sourceEncoding=UTF-8 \
//       -Dsonar.sources=. \
//       -Dsonar.host.url=https://sonarqube.spiderthings.co.in \
//       -Dsonar.login=sqp_f38f07a2419cefed3e7b35fa059010ca54aeda0d"
//               }
//               }
//     catch(Exception e) {
//           echo "FAILED ${e}"
//           currentBuild.result = 'FAILURE'
//           throw e
//         }               
//           }
//       }
// }

//           stage("Quality gate") {
//             steps {
//                 script {
//             try {
//                  sh 'sleep 10'
//                 timeout(time: 1, unit: 'MINUTES') {
//                 waitForQualityGate abortPipeline: true } 
//                 }
//   	     catch(Exception e) {
//           echo "FAILED ${e}"
//           currentBuild.result = 'FAILURE'
//           throw e
//         }                
                
//             } } }
            
      stage ('OWASP Dependency-Check Vulnerabilities') {
            steps {
                script {
                    def cause = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')
                }
                dependencyCheck additionalArguments: ''' 
                    -o "./" 
                    -s "./"
                    -f "ALL" 
                    --disableYarnAudit
                    --prettyPrint''', odcInstallation: 'owasp'

                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
        }     
        
    }
    
        stage('Container Running Verfication') {
            steps {
                script {
                    sh '''
                    echo login ECR
                    echo Docker Build
                    docker build -t test-app .
                    docker run -it -d  --name test-app -p 3080:3080 test-app
                    sleep 20
                    envA=$(curl -s -I localhost:3080 | grep HTTP/ | awk {'print $2'})
                    if [ "$envA" = "200" ]; then echo "all good" && docker rm -f test-app; else docker rm -f test-app && docker rmi -f test-app && docker images | grep none | awk '{ print $3; }' && exit 1; fi                
                    '''
                }
            } 
        }     
    
        stage('Deploying to Dev Enviroment') {
            steps {
                script {
                try {
                sh "docker logout"
                echo "Docker logout"
                // docker build -t test-app .
                sh '''
                docker logout
                aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 847280823661.dkr.ecr.us-east-1.amazonaws.com
                echo logged in ECR
                echo Delete Docker Images
                Commit=$(git log --oneline | head -n 1 | tr " " "-" | sed -e 's/([^()]*)//g' | sed 's/.$//')
                commit_id=$(git rev-parse --short=10 HEAD)                
                echo Docker Build
                
                echo Taging Image
                docker tag test-app:latest 847280823661.dkr.ecr.us-east-1.amazonaws.com/test-app:latest
                docker tag test-app:latest 847280823661.dkr.ecr.us-east-1.amazonaws.com/test-rollback:$Commit
                echo Remove old Image from ECR
                aws ecr batch-delete-image --repository-name test-app --image-ids imageTag=latest --region us-east-1
                echo Pushing image ECR
                docker push 847280823661.dkr.ecr.us-east-1.amazonaws.com/test-app:latest
                docker push 847280823661.dkr.ecr.us-east-1.amazonaws.com/test-rollback:$Commit
                              
                aws ecs update-service --cluster scrum-backend-cluster --service nodejs-backend --region us-east-1 --force-new-deployment 
                sleep 15
                aws ecs list-tasks --cluster scrum-backend-cluster --service nodejs-backend --region us-east-1              
                
                echo "==========Removing Docker images from Jenkins========="
                docker rmi -f 847280823661.dkr.ecr.us-east-1.amazonaws.com/test-app:latest
                docker rmi -f test-app:latest
                docker rmi -f test-rollback:$Commit
                '''
                } 
   	 catch(Exception e) {
          echo "FAILED ${e}"
          // slackSend (color: '#FF0000', message: "Failed at Deploying stage - ax-dev: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' " , channel: "jenkins-notification" )  
          currentBuild.result = 'FAILURE'
          throw e
        }  }             
                
            }
        }    
        
} }
