@Library('pipeline-library@master')
import com.genesys.jenkins.Service
import groovy.json.JsonSlurperClassic

properties([
        parameters([
                string(name: 'SERVICE_NAME', description: '', defaultValue: "${currentBuild.projectName}")                
        ]),
])

node('tester') {
    final String gitBase = 'Hello World'
    

    stage('print') {
        sh HelloWorld
    }

