podTemplate(label: 'mypod', containers: [
    containerTemplate(
        name: 'maven', 
        image: 'maven:3.3.9-jdk-8-alpine', 
        envVars: [envVar(key: 'MAVEN_SETTINGS_PATH', value: '/root/.m2/settings.xml')], 
        ttyEnabled: true, 
        command: 'cat'),
    containerTemplate(image: 'docker', name: 'docker', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.8.0', command: 'cat', ttyEnabled: true),
  ], volumes: [
    secretVolume(mountPath: '/root/.m2/', secretName: 'jenkins-maven-settings'),
    secretVolume(mountPath: '/home/jenkins/.docker', secretName: 'regsecret'),
    hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')
  ], imagePullSecrets: [ 'regsecret' ]) {

    node('mypod') {
        def projectNamespace = "${env.JOB_NAME}".tokenize('/')[0]
         container('kubectl') {
            stage('Configure Kubernetes') {
               sh "kubectl create namespace ${projectNamespace} || true"
               sh "kubectl create namespace ${projectNamespace}-production || true"
            }
         }

        git 'https://github.com/cd-pipeline/hello-world-service.git'
        container('maven') {
            stage('Build a project') {
                sh 'mvn -B clean install -DskipTests=true'
            }

            stage('Run tests') {
                try {
                    sh 'mvn -B clean install test'
                } finally {
                    junit 'target/surefire-reports/*.xml'
                }
            }

            stage('SonarQube Analysis') {
                try {
                    def srcDirectory = pwd();
                    def tmpDir = pwd(tmp: true)
                    dir(tmpDir) {
                        def scannerVersion = "2.8"
                        def localScanner = "scanner-cli.jar"
                        def scannerURL = "http://central.maven.org/maven2/org/sonarsource/scanner/cli/sonar-scanner-cli/${scannerVersion}/sonar-scanner-cli-${scannerVersion}.jar"
                        echo "downloading scanner-cli"
                        sh "curl -o ${localScanner} ${scannerURL} "
                        echo("executing sonar scanner ")
                        def projectKey = "${env.JOB_NAME}".replaceAll('/', "_")
                        sh "java -jar ${localScanner} -Dsonar.host.url=http://sonarqube:9000  -Dsonar.projectKey=${projectKey} -Dsonar.projectBaseDir=${srcDirectory} -Dsonar.java.binaries=${srcDirectory}/target/classes -Dsonar.sources=${srcDirectory}"
                    }

                } catch (err) {
                    echo "Failed to execute scanner:"
                    echo "Exception: ${err}"
                    throw err;
                }
            }

            stage('Deploy project to Nexus') {
                sh 'mvn -B -DskipTests=true package deploy'
                archiveArtifacts artifacts: 'target/*.jar'
            }
        }
        
        container('docker') {
            stage('Docker build') {
                sh 'docker build -t hello-world-service .'
                sh 'docker tag hello-world-service quay.io/zotovsa/hello-world-service'
                sh 'docker push quay.io/zotovsa/hello-world-service'
            }
        }

        container('kubectl') {
            stage('Deploy MicroService') {
               sh "kubectl delete deployment hello-world-service -n ${projectNamespace} || true"
               sh "kubectl delete service hello-world-service -n ${projectNamespace} || true"
               sh "kubectl create -f ./deployment/deployment.yml -n ${projectNamespace}"
               sh "kubectl create -f ./deployment/service.yml -n ${projectNamespace}"
               waitForAllPodsRunning('${projectNamespace}')
               waitForAllServicesRunning('${projectNamespace}')
            }
        }

        timeout(time: 3, unit: 'MINUTES') {
            input message: 'Deploy to Production?'
        }

        container('kubectl') {
           sh "kubectl delete deployment hello-world-service -n ${projectNamespace}-production || true"
           sh "kubectl delete service hello-world-service -n ${projectNamespace}-production || true"
           sh "kubectl create -f ./deployment/deployment.yml -n ${projectNamespace}-production"
           sh "kubectl create -f ./deployment/service.yml -n ${projectNamespace}-production"
           waitForAllPodsRunning('${projectNamespace}-production')
           waitForAllServicesRunning('${projectNamespace}-production')
        }
    }
}

def waitForAllPodsRunning(String namespace) {
    timeout(time: 3, unit: 'MINUTES') {
        while (true) {
            podsStatus = sh(returnStdout: true, script: "kubectl --namespace='${namespace}' get pods --no-headers").trim()
            def notRunning = podsStatus.readLines().findAll { line -> !line.contains('Running') }
            if (notRunning.isEmpty()) {
                echo 'All pods are running'
                break
            }
            sh "kubectl --namespace='${namespace}' get pods"
            sleep 10
        }
    }
}

def waitForAllServicesRunning(String namespace) {
    timeout(time: 3, unit: 'MINUTES') {
        while (true) {
            servicesStatus = sh(returnStdout: true, script: "kubectl --namespace='${namespace}' get services --no-headers").trim()
            def notRunning = servicesStatus.readLines().findAll { line -> line.contains('pending') }
            if (notRunning.isEmpty()) {
                echo 'All pods are running'
                break
            }
            sh "kubectl --namespace='${namespace}' get services"
            sleep 10
        }
    }
}
