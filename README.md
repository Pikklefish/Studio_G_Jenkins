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
