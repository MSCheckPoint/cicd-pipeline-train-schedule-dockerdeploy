pipeline {
    agent any
    environment {   
        /*Initialize Sourceguard SHIFTLEFT with CSPM Demo Portal*/ 
       SHIFTLEFT_REGION = 'eu1'
        
       CHKP_CLOUDGUARD_ID = credentials("CHKP_CLOUDGUARD_ID")
       CHKP_CLOUDGUARD_SECRET = credentials("CHKP_CLOUDGUARD_SECRET")
        
       SPECTRAL_DSN = credentials('spectral-dsn')
       gittoken1forspectral = credentials('gittoken1forspectral')
           
    }
     stages {

       stage('Cleaning Space') {
            steps {    
                echo 'removing builds'
                sh 'rm -rf /var/lib/jenkins/workspace/train-schedule_master/cicd-pipeline-train-schedule-dockerdeploy.tar'
                sh 'rm -rf /var/lib/jenkins/workspace/train-schedule_master/train-schedule_master@*'
                sh 'cd /var/lib/jenkins/workspace/train-schedule_master/ && ls'
            }
        }
              
         //spectral installation & scan                 
            stage('install Spectral') {
              steps {
                sh "curl -L 'https://app.spectralops.io/latest/x/sh?dsn=$SPECTRAL_DSN' | sh"
              }
            }
            stage('scan for issues') {
              steps {
                sh "$HOME/.spectral/spectral scan --ok  --include-tags base,audit"
              }
            }
            stage('CI/CD Hardening'){
               steps {
                //sh "$HOME/.spectral/spectral discover github --kind repo ."
                   sh "$HOME/.spectral/spectral discover github -k user MSCheckPoint --token gittoken1forspectral"
               }
                  }
            stage('Build') {
                steps {    
                    echo 'Running build automation'
                    sh './gradlew build --no-daemon'
                }
            }
             stage('SHIFTLEFT Source Code Scan') {            /*SAST scanning code for Vulnerabilities, Sensitive Content, Malicious IPs, Malicious URLS*/
        steps {           
           script {      
               try {
                  sh 'pwd'
                   sh 'shiftleft code-scan -s .'   

              } catch (Exception e) {
                  echo "Stage failed, but we continue!"  
                   }
                 }
              }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build("martyre37/cicd-pipeline-train-schedule-dockerdeploy")
                    app.inside {
                        sh 'echo $(curl localhost:8080)'
                    }
                }
            }
        }
        stage('SHIFTLEFT Container Image Scan') {                            /*Decomposing Layers of Container Image and scan Packages for Vulnerabilities*/
        steps {               
           script {      
               try {
                  sh 'docker save martyre37/cicd-pipeline-train-schedule-dockerdeploy > cicd-pipeline-train-schedule-dockerdeploy.tar'
                  sh 'export -p'
                 // sh 'shiftleft image-scan -i /var/lib/jenkins/workspace/train-schedule_master/cicd-pipeline-train-schedule-dockerdeploy.tar'
                   sh 'shiftleft image-scan -r -2002 -e 8a93d20b-5233-4b18-8361-4c5e533281e9 -i /var/lib/jenkins/workspace/train-schedule_master/cicd-pipeline-train-schedule-dockerdeploy.tar'
                   
              } catch (Exception e) {
                 // Optional Cleaning Workspace after failed build   
                //   sh rm -rf /var/lib/jenkins/workspace/train-schedule_master/train-schedule_master*
                       echo "Stage failed, but we continue! "
                   }
              }
          }
       }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                withCredentials([usernamePassword(credentialsId: 'CentOSprodForDocker', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip_CentOS_for_Docker \"docker pull martyre37/cicd-pipeline-train-schedule-dockerdeploy:${env.BUILD_NUMBER}\""
                        try {
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip_CentOS_for_Docker \"docker stop train-schedule\""
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip_CentOS_for_Docker \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip_CentOS_for_Docker \"docker run --restart always --name train-schedule -p 9090:8181 -d martyre37/cicd-pipeline-train-schedule-dockerdeploy:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
    stage('Secure WebApp in Production with Appsec') {   
        steps {    
            input 'Deploy with AppSec?'
            milestone(2)
            withCredentials([usernamePassword(credentialsId: 'CentOSprodForDocker', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                script {      
                  sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip_CentOS_for_Docker \"docker pull checkpoint/infinity-next-nginx\""
                  sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip_CentOS_for_Docker \"docker run -d --name=agent-container${env.BUILD_NUMBER} --ipc=host -v=/etc/cp/conf -v=/etc/cp/data -v=/var/log/nano_agent -it checkpoint/infinity-next-nginx /cp-nano-agent --token cp-553df5cf-36c7-4fe6-9334-60b303cba808ca0535bf-e41c-4078-8efc-865b40b992aa\""         
                  }
            }
          }
    }
}   
         post {
            // Clean after build
            always {
                cleanWs(cleanWhenNotBuilt: false,
                        deleteDirs: true,
                        disableDeferredWipeout: true,
                        notFailBuild: true,
                        patterns: [[pattern: '.gitignore', type: 'INCLUDE'],
                                   [pattern: '.propsfile', type: 'EXCLUDE']])
            }
        }
    }
