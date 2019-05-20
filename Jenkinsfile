@Library('pipeline-library@master')
import com.genesys.jenkins.Service
import groovy.json.JsonSlurperClassic

node('tester') {
    final String gitBase = 'Hello World'    
        
    stage('source') {
        git 
    }
    stage('print') {
        sh HelloWorld
    }

