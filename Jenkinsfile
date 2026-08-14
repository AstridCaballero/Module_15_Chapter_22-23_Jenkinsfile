pipeline {
    agent any
    environment {
        ANSIBLE_SERVER = "178.128.173.120"
    }
    stages {
        stage("copy files to ansible server") {
            steps {
                script {
                    echo "copying all necessary files to ansible control node"
                    sshagent(['ansible-server-key']) {
                        sh "scp -o StrictHostKeyChecking=no ansible/* root@${ANSIBLE_SERVER}:/root"

                        withCredentials([sshUserPrivateKey(credentialsId: 'ec2-server-key', keyFileVariable: 'keyFile', usernameVariable: 'user')]) {
                         sh 'scp $keyFile root@$ANSIBLE_SERVER:/root/ssh-key.pem'
                        }
                    }
                }
            }
        }
        stage ("execute ansible playbook") {
            steps {
                script {
                    echo "calling ansible playbook to configure EC2 instances"
                    def remote = [:]
                    remote.name = "ansible-server"
                    remote.host = env.ANSIBLE_SERVER
                    remote.allowAnyHosts = true

                    withCredentials([sshUserPrivateKey(credentialsId: 'ansible-server-key', keyFileVariable: 'keyFile', usernameVariable: 'user')]) {
                        remote.user = user
                        remote.identityFile = keyFile
                        sshCommand remote: remote, command: "ansible-playbook my-playbook.yaml"
                    }
                }
            }
        }
    }
} 
