pipeline{
    agent any
    tools{
        jdk 'JAVA_HOME'
        nodejs 'nodejs'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        TMDB_V3_API_KEY = credentials('tmdb-api-key')
        TRIVY_PATH = "C:\\Users\\gurra\\Downloads\\trivy_0.64.1_windows-64bit\\trivy.exe"
        DOCKER_PATH = "C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe"
        DOCKER_CONFIG = "${env.WORKSPACE}\\.docker"
        IMAGE = "netflix:latest"
        GIT_PATH = "C:\\Program Files\\Git\\cmd\\git.exe"
        POWERSHELL_PATH = "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe"
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/tarungurram29/DevSecOps-Project'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonarqube') {
                     withCredentials([string(credentialsId: 'netflix-token', variable: 'SONAR_TOKEN')]) {
                        bat """
                        %SCANNER_HOME%\\bin\\sonar-scanner \
                        -Dsonar.projectName=netflix \
                        -Dsonar.projectKey=netflix \
                        -Dsonar.host.url=http://localhost:9000 \
                        -Dsonar.login=%SONAR_TOKEN%
                        """
                     }
                }
            }
        }
        // stage("quality gate"){
        //   steps {
        //         script {
        //             waitForQualityGate abortPipeline: true
        //         }
        //     } 
        // }
        stage('Install Dependencies') {
            steps {
                bat "npm install"
            }
        }
        stage('Owasp scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'owasp'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        // stage('Trivy DB Download') {
        //      steps {
        //         bat '%TRIVY_PATH% --download-db-only --db-repository github.com/aquasecurity/trivy-db'
        //      }
        // }
        stage('trivy scan') {
            steps{
                bat '"%TRIVY_PATH%" fs --skip-db-update --skip-policy-update --skip-java-db-update . > trivyfs.txt'
            }
        }
        stage("Docker Build & Push"){
            steps{
                 withCredentials([
                     usernamePassword(credentialsId: 'docker', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS'),
                     string(credentialsId: 'tmdb-api-key', variable: 'TMDB_V3_API_KEY')]) {
                        script{
                            // Create a basic config.json without credential store
                            bat """
                            mkdir -p %DOCKER_CONFIG%
                            echo { "credsStore": "" } > %DOCKER_CONFIG%\\config.json
                            """
                             withEnv(["DOCKER_CONFIG=${env.WORKSPACE}\\.docker"]) {
                                bat "echo %DOCKER_PASS% | \"%DOCKER_PATH%\" login -u %DOCKER_USER% --password-stdin"
                                bat "\"%DOCKER_PATH%\" build --no-cache --build-arg TMDB_V3_API_KEY=%TMDB_V3_API_KEY% -t %IMAGE% . "
                                bat "\"%DOCKER_PATH%\" tag %IMAGE% fakefauxyy/%IMAGE%"
                                bat "\"%DOCKER_PATH%\" push fakefauxyy/%IMAGE%"
                             }
                        }
                     }
                }
        }
        stage("trivy"){
            steps{
                bat "%TRIVY_PATH% image fakefauxyy/%IMAGE% > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                bat "\"%DOCKER_PATH%\" run -d -p 8081:80 fakefauxyy/%IMAGE%"
            }
        }
        stage('updating yaml then push to github') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github_token',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    script{
                        def currentimage = "%IMAGE%"
                        def yamlimageline = readFile("Kubernetes/deployment.yml").readLines().find { it.contains("image:")}
                    
                    if(!yamlimageline.contains(currentimage)){
                        bat """
                        "%POWERSHELL_PATH%" -Command "(Get-Content Kubernetes\\deployment.yml) -replace 'image:.*', 'image: %IMAGE%' | Set-Content Kubernetes\\deployment.yml"
                        "%GIT_PATH%" config user.name "tarungurram29"
                        "%GIT_PATH%" config user.email "gurramtarun29@gmail.com"
                        "%GIT_PATH%" add Kubernetes\\deployment.yml
                        "%GIT_PATH%" remote set-url origin https://%GIT_USER%:%GIT_TOKEN%@github.com/%GIT_USER%/DevSecOps-Project
                        "%GIT_PATH%" commit -m "updated the deployment.yaml" || echo No changes to commit
                        "%GIT_PATH%" push origin main
                    """}
                    else{
                        echo "no push because nothing changed"
                    }
                    }
                }
            }
        }
        }
}
    // post {
    //  always {
    //     emailext attachLog: true,
    //         subject: "'${currentBuild.result}'",
    //         body: "Project: ${env.JOB_NAME}<br/>" +
    //             "Build Number: ${env.BUILD_NUMBER}<br/>" +
    //             "URL: ${env.BUILD_URL}<br/>",
    //         to: 'gurramtarun29@gmail.com',  
    //         attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
    //     }
    // }