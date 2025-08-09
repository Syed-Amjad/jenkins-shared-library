@Library('myLib') _
import org.example.Utils

pipeline {
    agent any
    stages {
        stage('Greeting') {
            steps {
                greet('Syed')
            }
        }
        stage('Parallel Tasks') {
            parallel {
                stage('Task A') {
                    steps {
                        echo Utils.shout("running task A")
                    }
                }
                stage('Task B') {
                    steps {
                        echo Utils.shout("running task B")
                    }
                }
            }
        }
    }
}
