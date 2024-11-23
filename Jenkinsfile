pipeline {
    agent any
    tools { 
        maven 'Maven_3_2_5'  
    }
    stages {
        stage('CompileandRunSonarAnalysis') {
            steps {	
                sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=final-capstone-lavanya_sample-code -Dsonar.organization=final-capstone-lavanya -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=2ad436d1a5c2d1e6a95d1301bdba9f5efe177e37'
            }
        }

        stage('RunSCAAnalysisUsingSnyk') {
            steps {
                script {
                    try {
                        withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                            sh 'mvn snyk:test -fn'
                        }
                    } catch (Exception e) {
                        // If Snyk analysis fails, create a JIRA ticket
                        echo "Snyk analysis failed. Creating JIRA ticket..."
                        
                        // JIRA Issue Creation Logic
                        def jiraServer = 'JIRA_SERVER' 
                        def testIssue = [
                            fields: [
                                project: [id: '10000'], 
                                summary: 'SCA Analysis Failed - Action Required',
                                description: "SCA analysis failed during Jenkins pipeline execution. Error: ${e}",
                                issuetype: [name: 'Bug'], 
                                assignee: [username: 'Lavanya Pidikiti'] 
                            ]
                        ]
                        def response = jiraNewIssue(issue: testIssue, site: jiraServer)
                        
                        echo "JIRA Ticket Created: ${response.data.key}"
                    }
                }
            }
        }
	   
        stage('Build') { 
            steps { 
                withDockerRegistry([credentialsId: "dockerlogin", url: ""]) {
                    script {
                        app = docker.build("myimg")
                    }
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    docker.withRegistry('https://515911444422.dkr.ecr.us-east-1.amazonaws.com', 'ecr:us-east-1:aws-credentials') {
                        app.push("latest")
                    }
                }
            }
        }
	   
        stage('wait_for_testing') {
            steps {
                sh 'pwd; sleep 180; echo "Application has been deployed on K8S"'
            }
        }
	   
        stage('RunDASTUsingZAP') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    sh 'zap.sh -cmd -quickurl http://$(kubectl get services/easybuggy --namespace=devsecops -o json | jq -r ".status.loadBalancer.ingress[] | .hostname") -quickprogress -quickout ${WORKSPACE}/zap_report.html'
                    archiveArtifacts artifacts: 'zap_report.html'
                }
            }
        }
    }
}
