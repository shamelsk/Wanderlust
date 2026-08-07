@Library('Shared') _
pipeline {
    agent {label 'Node'}
    
    environment{
        SONAR_HOME = tool "Sonar"
    }
    
    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: '', description: 'Setting docker image for latest push')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: '', description: 'Setting docker image for latest push')
    }
    
    stages {
        stage("Validate Parameters") {
            steps {
                script {
                    if (params.FRONTEND_DOCKER_TAG == '' || params.BACKEND_DOCKER_TAG == '') {
                        error("FRONTEND_DOCKER_TAG and BACKEND_DOCKER_TAG must be provided.")
                    }
                }
            }
        }
        stage("Workspace cleanup"){
            steps{
                script{
                    cleanWs()
                }
            }
        }
        
        stage('Git: Code Checkout') {
            steps {
                script{
                    code_checkout("https://github.com/shamelsk/Wanderlust.git","main")
                }
            }
        }
        
        stage("Trivy: Filesystem scan"){
            steps{
                script{
                    trivy_scan()
                }
            }
        }

        stage("OWASP: Dependency check"){
            steps{
                script{
                    owasp_dependency()
                }
            }
        }
        
        stage("SonarQube: Code Analysis"){
            steps{
                script{
                    sonarqube_analysis("Sonar","wanderlust","wanderlust")
                }
            }
        }
        
        stage("SonarQube: Code Quality Gates"){
            steps{
                script{
                    sonarqube_code_quality()
                }
            }
        }
        
        stage('Exporting environment variables') {
            parallel{
                stage("Backend env setup"){
                    steps {
                        script{
                            dir("Automations"){
                                sh "bash updatebackendnew.sh"
                            }
                        }
                    }
                }
                
                stage("Frontend env setup"){
                    steps {
                        script{
                            dir("Automations"){
                                sh "bash updatefrontendnew.sh"
                            }
                        }
                    }
                }
            }
        }
        
        stage("Docker: Build Images"){
            steps{
                script{
                        dir('backend'){
                            docker_build("wanderlust-backend-beta","${params.BACKEND_DOCKER_TAG}","shamel1012")
                        }
                    
                        dir('frontend'){
                            docker_build("wanderlust-frontend-beta","${params.FRONTEND_DOCKER_TAG}","shamel1012")
                        }
                }
            }
        }
        
        stage("Docker: Push to DockerHub"){
            steps{
                script{
                    docker_push("wanderlust-backend-beta","${params.BACKEND_DOCKER_TAG}","shamel1012") 
                    docker_push("wanderlust-frontend-beta","${params.FRONTEND_DOCKER_TAG}","shamel1012")
                }
            }
        }
    }
    post {
    success {
        archiveArtifacts artifacts: '*.xml', followSymlinks: false

        build job: "Wanderlust-CD", parameters: [
            string(name: 'FRONTEND_DOCKER_TAG', value: "${params.FRONTEND_DOCKER_TAG}"),
            string(name: 'BACKEND_DOCKER_TAG', value: "${params.BACKEND_DOCKER_TAG}")
        ]
    }

    failure {
        emailext(
            attachLog: true,
            from: 'shamelsk10@gmail.com',
            to: 'shamelsk10@gmail.com',
            subject: "Wanderlust CI Pipeline Failed - ${currentBuild.currentResult}",
            mimeType: 'text/html',
            body: """
            <html>
            <body>
                <h2 style="color:red;">CI Pipeline Failed</h2>

                <p><b>Project:</b> ${env.JOB_NAME}</p>

                <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>

                <p><b>Status:</b> ${currentBuild.currentResult}</p>

                <p><b>Build URL:</b>
                <a href="${env.BUILD_URL}">
                    ${env.BUILD_URL}
                </a></p>

                <p>Please check the Jenkins console log to identify the failed stage.</p>
            </body>
            </html>
            """
        )
    }
}
