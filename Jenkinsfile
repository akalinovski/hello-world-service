podTemplate(label: 'mypod', containers: [
    containerTemplate(
        name: 'maven', 
        image: 'maven:3.3.9-jdk-8-alpine', 
        envVars: [envVar(key: 'MAVEN_SETTINGS_PATH', value: '/root/.m2/settings.xml')], 
        ttyEnabled: true, 
        command: 'cat'),
    containerTemplate(image: 'docker', name: 'docker', command: 'cat', ttyEnabled: true)
  ], volumes: [
    secretVolume(mountPath: '/root/.m2/', secretName: 'jenkins-maven-settings'),
    hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')
  ]) {

    node('mypod') {
        git 'https://github.com/cd-pipeline/health-check-service.git'
        container('maven') {
            stage('Build a Maven project') {
                sh 'mvn -B clean install -DskipTests'
            }

            stage('Echo') {
                sh 'mvn -B package deploy'
            }
        }
        
        container('docker') {
            stage('Docker build') {
                sh 'docker build -t health-check-service .'
            }
        }
    }
}
