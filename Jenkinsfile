pipeline {
    agent any
    stages {
        stage("copy files to ansible server") {
            steps {
                script {
                    echo "copying all necessary files to ansible control node"
                    sshagent(['ansible-server-key']) {
                        sh "scp -o StrictHostKeyChecking=no ansible/* root@178.128.173.120:/root"

                        withCredentials([sshUserPrivateKey(credentialsId: 'ec2-server-key', keyFileVariable: 'keyFile', usernameVariable: 'user')]) {
                         sh 'scp $keyFile root@178.128.173.120:/root/ssh-key.pem'
                        }
                    }
                }
            }
        }
    }
} 
