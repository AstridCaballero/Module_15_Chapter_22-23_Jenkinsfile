![img.png](img.png)
## Module 15 - Configuration Management with Ansible

### Chapter 23 - Project: Run Ansible from Jenkins Pipeline - Part 3

Notes from chapter 23 were taken in notes app in freestyle, and it includes a mix of screenshots of TWN bootcamp as well as
screenshots of my own work on the terminal and browser. I used AI and I have added that research to my notes.

Notes have been exported into pdf format to be able to get the screenshots (avoiding retaking screenshots which is time-consuming)

You can find original notes for
- Chapter 23 [here](Module%2015%20-%20chapter%2023_notes.pdf).

### Description Module 15 - chapter 21 to 23: Run Ansible from Jenkins Pipeline
This DEMO executes Ansible playbooks from Jenkins pipeline to configure two EC2 instances. 

The DEMO involved three chapters:  chapter 21, 22 and 23, below are the three branches (one belongs to one repo and the other two branches to another repo) related to each chapter.

https://github.com/AstridCaballero/Module_15_Ansible/tree/Module_15/Ansible_chapter-21

Java app branches:

https://github.com/AstridCaballero/Module_15_Chapter_22-23_Jenkinsfile/tree/Module_15/Ansible_chapter-22

https://github.com/AstridCaballero/Module_15_Chapter_22-23_Jenkinsfile/tree/Module_15/Ansible_chapter-23


### Chapter 21

For this DEMO we need:
- Jenkins server
- Ansible server
- Java-app project
- AWS EC2 instances


#### Ansible server
- Create ansible server in a DigitalOcean droplet and assign an ssh key to it.
- Configure ansible server -> using the ssh key linked to the newly created ansible server ssh into the server from my local machine and install:
    - Ansible
    - Boto3 -> droplet already has boto3 installed
    - Botocore -> droplet already has botocore installed

	Ansible is a python program and it doesn’t know how to talk to a specific API. We will work with EC2 instances and want to 
	get their IP addresses dynamically so we need Ansible to talk to AWS API. 

	We need to install an Ansible Collection (or package) called ‘amazon.aws’ . This Collection has modules and plugins. 
	Those modules and plugins are tools that call boto3.

	Boto3 knows how to connect to AWS API. This is the reason why we need Boto3 and botocore (authentication, reties, etc)

- Configure the AWS credentials (access key and secret access key) inside the Ansible server. We need them for Boto3 to use them and auth to AWS API to make the connection.

		access key + secret access key -> AWS API

- Ansible also will ssh into the EC2 instances to execute the playbook that will configure them, for that it will need a pem file but this information will be passed via Jenkinsfile.
	
		AWS key pair (pem) -> ssh into EC2 server

### Chapter 22

#### AWS EC2 instances
- Create 2 EC2 instances from the AWS console.
- Create a new key pair from AWS console -> returns a pem file. Ansible will use the pem file to ssh into the EC2 instances to configure them when required. This key pair will be pass to Ansible via Jenkins file as already mentioned.


#### Jenkins server
- Create credentials
    - Ansible server credential -> ‘ansible-server-key’ (the key is stored in my laptop in the /.ssh folder as id_ed25519, so I display it in my terminal and then copy and paste it into Jenkins)
    - AWS credential -> ‘ec2-server-key’ (AWS pem file that I downloaded in my local machine, so I display it in my terminal and then copy and paste it into Jenkins)
- Create a simple Jenkins Pipeline like the one from Module 8 - chapter 4 and link the Java-app repo to the pipeline.

#### Java-app project
- Create directory ‘ansible’
    - Add files:
        - ansible.cfg -> make sure to set 
            - ‘inventory’ to ‘inventory_aws_ec2.yaml’
            - ‘enable_plugins’ to ‘aws_ec2’
            - ‘private_key_file’ to the path of the AWS key pair in Ansible server->  ‘~/ssh-key.pem’ (Jenkinsfile will copy the key pair in Ansible server in time for this ansible.cfg to use it)
        - inventory_aws_ec2.yaml -> make sure to set
            - ‘plugin’ to ‘aws_ec2’ -> in order to fetch the EC2’s IP addresses dynamically via the plugin.
        - my-playbook.yaml to install in the EC2 instances:
            - Docker
            - Docker-compose
- Create Jenkinsfile file
    - Add stage “copy files to ansible server". In this stage we will write logic to copy 4 files into the Ansible server:
        - Ansible configuration -> ansible.cfg
        - ‘hosts’ or ‘inventory_aws_ec2.yaml’ file -> we will use the latter because we will get the EC2 instances’ IP addresses dynamically.
        - Playbook -> ‘my-playbook.yaml’
        - Add logic for Jenkins to ssh into Ansible server
            - Use sshagent Jenkins plugin
                - Pass argument -> the Jenkins credential holding the ansible server key called ‘ansible-server-key’
                - Add a shell command to securely copy the files inside the ‘ansible’ folder (3 of the 4 files that Ansible server needs) and to don’t ask about host key ‘StrictHostKeyChecking=no’
			  
			 	sshagent does not open a connection to anything, it loads the private key and makes it reachable for the shell 
			  	commands that we run inside the block. There is a new connection each time we run a sh command, so we 
				need to pass ‘<server_user>@<server_ip>’ 

			 	 and we also pass the location to store the file in the remote server which is Ansible server -> ‘/root’

				 and The final command is:
					sh "scp -o StrictHostKeyChecking=no ansible/* root@${ANSIBLE_SERVER}:/root"

                - Fetch the AWS credential ‘ec2-server-key’ using ‘withCredentials’ -> this will provide the content of the credential which is the key and the user
                - Add a shell command to securely copy the temp file created by ‘withCredentials’ from credential ‘ec2-server-key’ which was stored in variable ‘keyFile’. 
			
			  	sshagent does not open a connection to anything, it loads the private key and makes it reachable for the shell
				commands that we run inside the block. There is a new connection each time we run a sh command, so we 
				need to  pass ‘<server_user>@<server_ip>’ 

			  	and we also pass the location to store the file in the remote server which is Ansible server 
				-> ‘/root/ssh-key.pem’

			  	and The final command is:
					sh 'scp $keyFile root@$ANSIBLE_SERVER:/root/ssh-key.pem'

		The logic in this stage will get Jenkins to copy the four files we mentioned earlier that Ansible server needs to configure
		the two EC2 servers.

    - Add stage ‘execute ansible playbook’. In this stage we will write logic to get Jenkins to execute commands in Ansible server where finally the EC2 instances get configured by Ansible playbook ‘my-playbook.yaml’.
        - Add logic for Jenkins to ssh into Ansible server
            - Use sshComand Jenkins plugin (an alternative to sshagent)
                - Create an object to store the ansible server’s information that Jenkins needs to shh into the remote server. We call it ‘remote’ and set the below attributes to it:
                    - name -> label for log output
                    - host -> remote server IP
                    - allowAnyHosts -> don’t ask about host key
                    - User -> user name from credential ‘ansible-server-key’ fetched using ‘withCredentials’
                    - identityFile -> Path to the private key temp file fetched using ‘withCredentials’
                - Use sshCommand inside ‘withCredentials’ block we need to pass two arguments:
                    - remote ->  object with the information gathered from the server -> the objet is called ‘remote’
                    - command -> the command to execute the playbook -> “ansible-playbook my-playbook.yaml"
					
					final code using sshCommand:
						sshCommand remote: remote, command: "ansible-playbook my-playbook.yaml"

### Test
Push java-app project to GitHub and then run Jenkins Pipeline:
- Originally it failed mainly due to configuration inconsistencies in ‘my-playbook.yaml’ file so I fixed the issues there.
- Then pipeline worked

### Chapter 23

#### Optimisations
- Reduce duplication
    - Create env var to store Ansible’s IP address
- Create a script to configure Ansible server so we configure ansible server via Jenkins instead of manually like we did in this DEMO. So the script installs ansible and boto3 (by installing boto3 then botocore gets installed as well)
	
  We can then add sshScript (another step provided by the plugin ‘ssh pipeline steps’) in the stage ‘execute ansible playbook’
  inside the ‘withCredentials’ block allowing for sshCommand to run successfully.

- We added the AWS configuration credential manually to Ansible server at the beginning of the DEMO. We can also store this credentials in a Jenkins credential and then add another ‘withCredentials’ block in the stage ‘copy files to ansible server’ and:
    - Execute a shell command to ssh and create a directory called ‘.aws’ -> /root/.aws
    - Securely copy the key from the temp file created by ‘withCredentials’ into /root/.aws/credentials
			

I learnt what credentials I need to create in Jenkins and how to use them in Jenkinsfile to get:
- Jenkins to shh into Ansible server.
- Jenkins to connect Ansible to AWS API to get the EC2 instances IP addresses dynamically. 
- Jenkins to execute Ansible inside Ansible server and configure two EC2 instances.

I also, learnt how to use sshCommand plugin. And I reinforce the understanding of Ansible, its collections, plugins (aws_ec2) and the flow to reach boto3 and connect to AWS API. This thought me that in a setting like this Ansible will mainly ssh into the EC2 servers and rarely connect to AWS API.


