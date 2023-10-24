pipeline {
    agent none
    environment {
        DOCKERHUB_CREDENTIALS = credentials('DockerLogin')
        SNYK_CREDENTIALS = credentials('SnykToken')
    }
    stages {
        stage('Secret Scanning Using Trufflehog') {
            agent {
                docker {
                    image 'trufflesecurity/trufflehog:latest'
                    args '-u root --entrypoint='
                }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'trufflehog filesystem . --exclude-paths trufflehog-excluded-paths.txt --fail --json > trufflehog-scan-result.json'
                }
                sh 'cat trufflehog-scan-result.json'
                archiveArtifacts artifacts: 'trufflehog-scan-result.json'
            }
        }
        stage('Build') {
            agent {
              docker {
                  image 'node:lts-buster-slim'
              }
            }
            steps {
                sh 'npm install'
            }
        }
        stage('Test') {
            agent {
              docker {
                  image 'node:lts-buster-slim'
              }
            }
            steps {
                sh 'npm run test'
            }
        }
        stage('SCA Snyk Test') {
            agent {
              docker {
                  image 'snyk/snyk:node'
                  args '-u root --network host --env SNYK_TOKEN=$SNYK_CREDENTIALS_PSW --entrypoint='
              }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'snyk test > snyk-scan-report.txt'
                }
                sh 'cat snyk-scan-report.txt'
                archiveArtifacts artifacts: 'snyk-scan-report.txt'
            }
        }
        stage('SCA Retire Js') {
            agent {
              docker {
                  image 'node:lts-buster-slim'
              }
            }
            steps {
                sh 'npm install retire'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh './node_modules/retire/lib/cli.js --outputpath retire-scan-report.txt'
                }
                sh 'cat retire-scan-report.txt'
                archiveArtifacts artifacts: 'retire-scan-report.txt'
            }
        }
        stage('SAST Snyk') {
            agent {
              docker {
                  image 'snyk/snyk:node'
                  args '-u root --network host --env SNYK_TOKEN=$SNYK_CREDENTIALS_PSW --entrypoint='
              }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'snyk code test > snyk-sast-report.txt'
                }
                sh 'cat snyk-scan-report.txt'
                archiveArtifacts artifacts: 'snyk-sast-report.txt'
            }
        }
        stage('SAST SonarQube') {
            agent {
              docker {
                  image 'sonarsource/sonar-scanner-cli:latest'
                  args '--network host -v ".:/usr/src" --entrypoint='
              }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'sonar-scanner -Dsonar.projectKey=NodeGoat -Dsonar.qualitygate.wait=true -Dsonar.sources=. -Dsonar.host.url=http://localhost:9000 -Dsonar.token=sqp_2f839e4c5ea3eb6387d2c29ae5776aa7dd0ec327' 
                }
            }
        }
        stage('Build Docker Image and Push to Docker Registry') {
            agent {
                docker {
                    image 'docker:dind'
                    args '--user root --network host -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh 'docker build -t xenjutsu/nodegoat:0.1 .'
                sh 'docker push xenjutsu/nodegoat:0.1'
            }
        }
        stage('Deploy Docker Image') {
            agent {
                docker {
                    image 'kroniak/ssh-client'
                    args '--user root --network host'
                }
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: "DeploymentSSHKey", keyFileVariable: 'keyfile')]) {
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no jenkins@192.168.0.102 "echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin"'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no jenkins@192.168.0.102 docker pull xenjutsu/nodegoat:0.1'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no jenkins@192.168.0.102 docker rm --force nodegoat'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no jenkins@192.168.0.102 docker run -it --detach -p 4000:4000 --name nodegoat --network host xenjutsu/nodegoat:0.1'
                }
            }
        }
//        stage('DAST Nuclei') {
//            agent {
//                docker {
//                    image 'projectdiscovery/nuclei'
//                    args '--user root --network host --entrypoint='
//                }
//            }
//            steps {
//                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
//                    sh 'nuclei -u http://localhost:4000 -j > nuclei-report.json'
//                    sh 'cat nuclei-report.json'
//                }
//                archiveArtifacts artifacts: 'nuclei-report.json'
//            }
//        }
//        stage('DAST OWASP ZAP') {
//            agent {
//                docker {
//                    image 'owasp/zap2docker-stable:latest'
//                    args '-u root --network host -v /var/run/docker.sock:/var/run/docker.sock --entrypoint= -v .:/zap/wrk/:rw'
//                }
//            }
//            steps {
//                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
//                    sh 'zap-full-scan.py -t http://192.168.1.84:4000 -r zapfull.html -x zapfull.xml'
//                }
//                sh 'cp /zap/wrk/zapfull.html ./zapfull.html'
//                sh 'cp /zap/wrk/zapfull.xml ./zapfull.xml'
//                archiveArtifacts artifacts: 'zapfull.html'
//                archiveArtifacts artifacts: 'zapfull.xml'
//            }
//        }
    }
//    post {
//        always {
//            node('built-in') {
//                sh 'curl -X POST http://localhost:8080/api/v2/import-scan/ -H "Authorization: Token 4352361ac0640d6cb3284e5354f194fc89344c14" -F "scan_type=Trufflehog Scan" -F "file=@./trufflehog-scan-result.json;type=application/json" -F "engagement=1"'
//                sh 'curl -X POST http://localhost:8080/api/v2/import-scan/ -H "Authorization: Token 4352361ac0640d6cb3284e5354f194fc89344c14" -F "scan_type=Nuclei Scan" -F "file=@./nuclei-report.json;type=application/json" -F "engagement=1"'
//                sh 'curl -X POST http://localhost:8080/api/v2/import-scan/ -H "Authorization: Token 4352361ac0640d6cb3284e5354f194fc89344c14" -F "scan_type=ZAP Scan" -F "file=@./zapfull.xml;type=text/xml" -F "engagement=1"'
//            }
//        }
//   }
}
