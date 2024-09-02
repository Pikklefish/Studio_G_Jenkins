# Jenkins CI/CD Deployment Documentation
This project is part of a preliminary research and documentation for the use of engineers at Studio Galilei R&D Division (Create G)

## Deployment on Linux Environment using Docker

[refer to this link for different environment set ups](https://www.jenkins.io/doc/book/installing/docker/)

**Resources Used**
- Portainer.io: easy Docker container management
- Xshell7: terminal to access internal server

Access the terminal execute the following commands. 

Create a bridge network in Docker

```docker network create jenkins ```


Next create a Docker image for the Official Jenkins image


```
docker run --name jenkins-docker --rm --detach \
  --privileged --network jenkins --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 \
  docker:dind --storage-driver overlay2
```

Up to here you have created the official Jenkins image and creating a container from this image will cause no problems, however,
if you want to have `Docker-CLI` or `Blueocean` please continue below.
Create a `Dockerfile` to configure Docker settings
```
touch Dockerfile
```
`i` to enter insert mode, type out the Dockerfile, `:wq` to exit insert mode and press `enter`. The Docker file will contain the
necessary libraries to run the Jenkins pipeline. From the base configuration `FROM jenkins/jenkins:2.462.1-jdk17`
we needed extra tools such as `sshpass` to execute this pipeline.
```
FROM jenkins/jenkins:2.462.1-jdk17
USER root
RUN apt-get update && \
    apt-get install -y sshpass curl gnupg lsb-release && \
    curl -fsSL https://download.docker.com/linux/debian/gpg | tee /usr/share/keyrings/docker-archive-keyring.asc && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.asc] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && \
    apt-get install -y docker-ce-cli && \
    apt-get install -y openssh-client && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow"
```

Run the build and give it a meaningful name (ie . `myjenkins-blueocean:2.462.1-1`). This code will pull the libraries and install
them.

```docker build -t myjenkins-blueocean:2.462.1-1 .```

Finally create the Docker container from the image. 
```
docker run --name jenkins-blueocean --restart=on-failure --detach \
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
  --publish 8932:8080 --publish 50322:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  --volume /home/varo/geoje-dev-key.pem:/var/jenkins_home/key.pem:ro \
  myjenkins-blueocean:2.462.1-1
```
You can see that the host is docker:2376 but since we are adding the plugins on top of the official Jenkins image we are 
publishing it newly to 8080 (1234:5678, 1234 is the port that’s accessible externally and 5678 is an internal port).

Proceed and access Jenkins on `localhost:8932`

## Unlocking Jenkins (Initial Admin Password)
Attain the initial admin password using this command ```docker exec [container_name/ID] cat /var/jenkins_home/secrets/initialAdminPassword```

## Trouble Shooting
When you access your Jenkins and see this error
```
Some plugins could not be loaded due to unsatisfied dependencies. Fix these issues and restart Jenkins to re-enable these plugins.

Dependency errors:

Token Macro Plugin (400.v35420b_922dcb_)
Plugin is missing: json-path-api (2.8.0-5.v07cb_a_1ca_738c)
Some of the above failures also result in additional indirectly dependent plugins not being able to load.

Indirectly dependent plugins:

GitHub Pipeline for Blue Ocean (1.27.14)
Failed to load: Pipeline implementation for Blue Ocean (blueocean-pipeline-api-impl 1.27.14)
Git Pipeline for Blue Ocean (1.27.14)
Failed to load: Pipeline implementation for Blue Ocean (blueocean-pipeline-api-impl 1.27.14)
GitHub plugin (1.40.0)
Failed to load: Token Macro Plugin (token-macro 400.v35420b_922dcb_)
REST Implementation for Blue Ocean (1.27.14)
Failed to load: Favorite (favorite 2.221.v19ca_666b_62f5)
Events API for Blue Ocean (1.27.14)
Failed to load: Pipeline implementation for Blue Ocean (blueocean-pipeline-api-impl 1.27.14)
Blue Ocean Pipeline Editor (1.27.14)
Failed to load: Pipeline implementation for Blue Ocean (blueocean-pipeline-api-impl 1.27.14)
Bitbucket Pipeline for Blue Ocean (1.27.14)
Failed to load: Pipeline implementation for Blue Ocean (blueocean-pipeline-api-impl 1.27.14)
GitHub Branch Source Plugin (1797.v86fdb_4d57d43)
Failed to load: GitHub plugin (github 1.40.0)
Blue Ocean (1.27.14)
Failed to load: Bitbucket Pipeline for Blue Ocean (blueocean-bitbucket-pipeline 1.27.14)
Favorite (2.221.v19ca_666b_62f5)
Failed to load: Token Macro Plugin (token-macro 400.v35420b_922dcb_)
Pipeline implementation for Blue Ocean (1.27.14)
Failed to load: REST Implementation for Blue Ocean (blueocean-rest-impl 1.27.14)
```
Follow this step
1. The problem arises because you are missing the `json-path-api` plugin
2. You simple go to `Manage Jenkin > Plugins > Available Plugins` and look for the `json-path-api`
3. Or... go to [JSON Path API](https://plugins.jenkins.io/json-path-api/) and direct install
4. Download the latest version
5. Navigate to `Plugin > Advanced Setting` upload the downloaded file directly
6. Manually restart the Jenkins containers (I did that in Portainer.io)


# Building the Pipeline
The purpose of this pipeline is to bring files from Sonatype Nexus repository to an internal SFTP server.

First the credentials for Nexus and SFTP server needs to be set up. Nexus is accessible using `username:password` and 
the SFTP server is accessible using a `key.pem` file. This documentation will go over how to set each of those up.

## With `Username:Password`
To set up the `username:password` navigate to `Mange Jenkins > Credentials > (select global) > Add Credentials `. Select 
kind as `Username with password`, scope as `Global(Jenkins, nodes, ...)`, insert username/password and create. Back at the
`Credentials` page you will be able to see Credentials ID for each credential, can be useful later on.

Now create a pipeline with a name that you want from the main page. In the job, on the left column select `Pipeline Syntax`.
For `Sample step > withCredentials: Bind Credentials to variables`, select a variable name (ie `Variable > NEXUS_Credentials`)
and select the credentials you want to bind your variable to. Generate pipeline script, copy that and paste it into your pipeline.
It should look something like this:
```
stage('Pull the file off Nexus') {
        withCredentials([usernameColonPassword(credentialsId: '5771e4b4-a87b-4a7b-b536-2a68bdc8caa9', variable: 'NEXUS_CREDENTIALS')]) {
           script ...
        }
    }
```
As a side note I will also include how ot use username and password with an SFTP server. The username and password were extracted.
```
 stage('Upload file(s) to server') {
            withCredentials([usernameColonPassword(credentialsId: 'ba0bd89f-af69-40dc-af02-842b82fc02be', variable: 'FTP_CREDENTIALS')]) {

            // Extract the credentials in Groovy
            def ftpUser = "${FTP_CREDENTIALS.split(':')[0]}"
            def ftpPass = "${FTP_CREDENTIALS.split(':')[1]}"

            // Upload the file using sshpass with sftp
            sh script: """
            sshpass -p ${ftpPass} sftp -oBatchMode=no ${ftpUser}@192.168.15.170 <<EOF
            cd /home/peter/deploy
            put ${SAVE_FILE_NAME}
            EOF
            """
        }

    }
```

## With `key.pem` file
To use `key.pem` to gain access you need to first load the `key.pem` file to either your local device (if you are working just on your computer)
or to your company's remote server. You will then add that `key.pem` file to your Docker container as a volume when you are 
running your Docker container. This is what needs to be added: ```--volume /home/varo/geoje-dev-key.pem:/var/jenkins_home/key.pem:ro \```
```/home/varo/geoje-dev-key.pem``` is the location of the `key.pem` on my server and ```var/jenkins_home/key.pem:ro```
is where I am mapping it to on Docker.

Edited version of the Docker run command
```
docker run --name jenkins-blueocean --restart=on-failure --detach \
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
  --publish 8932:8080 --publish 50322:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  --volume /home/varo/geoje-dev-key.pem:/var/jenkins_home/key.pem:ro \
  myjenkins-blueocean:2.462.1-1
```

You will then be able to use the key to gain SFTP access. Below is the sample code using the key
```
    stage('Backup and Upload file(s) to server') {
        SERVER_LIST.each { server ->
            sh """
            # Create backup filename by appending '_backup' to the original file name
            BACKUP_FILE_NAME="${SAVE_FILE_NAME}_backup"

            # Check if the file exists on the remote server, if it does, back it up
            ssh -i /var/jenkins_home/key.pem -o StrictHostKeyChecking=no ${USERNAME}@${server[0]} "[ -f ${server[1]}/${SAVE_FILE_NAME} ] && cp -pf ${server[1]}/${SAVE_FILE_NAME} ${server[1]}/\${BACKUP_FILE_NAME} || echo 'No existing file to backup'"

            # Upload the new WAR or JAR file to the server
            scp -i /var/jenkins_home/key.pem -o StrictHostKeyChecking=no ${SAVE_FILE_NAME} ${USERNAME}@${server[0]}:${server[1]}
            """
        }
    }
```


For the final pipeline I added in features to satisfy the spec required. For example, the variables like the `SAVE_FILE_NAME` 
have been moved outside the node to be used as a global variable across stages. The second stage also not only just uploads it 
to a SFTP server but also saves the old file as `{file_name}_backup` in case we need to perform a rollback.
```
/////////FINAL VERSION
def NEXUS_LINK = "{NEXUS_LINK}"  // path to file on NEXUS repository
def SAVE_FILE_NAME = "{FILE_NAME}"  // name to save file as
def USERNAME = "{USER_ID}" // user variable inside the node
def SERVER_LIST = [
    ['{SFTP_SREVER}', '{DIRECTORY_PATH}']
]

node {
    stage('Import from NEXUS') {
        withCredentials([usernameColonPassword(credentialsId: 'ee15f893-1728-450b-9556-2647a12f856d', variable: 'NEXUS_CREDENTIALS')]) {
            sh script: """
            curl -u \$NEXUS_CREDENTIALS -o ${SAVE_FILE_NAME} ${NEXUS_LINK}
            """
        }
    }

    stage('Backup and Upload file(s) to SFTP') {
        SERVER_LIST.each { server ->
            sh """
            # Create backup filename to the original file name
            BACKUP_FILE_NAME="${SAVE_FILE_NAME}_backup"

            # Check if the file exists on the remote server, if it does, back it up
            ssh -i /var/jenkins_home/key.pem -o StrictHostKeyChecking=no ${USERNAME}@${server[0]} "[ -f ${server[1]}/${SAVE_FILE_NAME} ] && cp -pf ${server[1]}/${SAVE_FILE_NAME} ${server[1]}/\${BACKUP_FILE_NAME} || echo 'No existing file to backup'"

            # Upload the new WAR or JAR file to the server
            scp -i /var/jenkins_home/key.pem -o StrictHostKeyChecking=no ${SAVE_FILE_NAME} ${USERNAME}@${server[0]}:${server[1]}
            """
        }
    }
}

```
