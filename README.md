# CI Installation

## Requirements
1. Docker
2. Jenkins

## Install Docker

### Linux
1. See [Install Docker Engine on Linux](https://docs.docker.com/engine/installation/linux/).
2. if you want to use docker without `sudo`:
    - Remember to add yourself to the `docker` group. `sudo groupadd docker && sudo gpasswd -a $USER docker`
    - Restart the daemon:
        - In Ubuntu: `sudo service docker restart`
        - Other distributions: `sudo systemctl restart docker`

### Mac OS X
1. See [Getting Started with Docker for Mac](https://docs.docker.com/docker-for-mac/).

## Basic Docker Usage

Note:  All commands should run with sudo permission unless you have set the docker group.
### Terminologies
1. Container: VM-like instance running processes.
2. Image: Disk image for containers. Defined by a Dockerfile. You can get a lot of images from [Docker Hub](https://hub.docker.com/)
3. Volume: A host directory that can be mount into container directory.

### Container Management
1. Show current running containers: `docker ps`
2. Show all containers: `docker ps -a`. This includes exited dockers.
3. Show available docker images: `docker images`
4. Show all volumes: `docker volume ls`
5. To delete a container / image / volume:
    - Use 1. to 4. to get ID / NAME (something like `9e24d7d5a3be` or `jenkins_workspace`).
    - Container: `docker rm [ID]`
    - Image: `docker rmi [ID]`
    - Volume: `docker volume rm [NAME]`
6. Tip: to remove all unused containers: `docker ps -a | awk '{print $1}' | xargs docker rm`

### Running a Container

```bash
$ docker run [Options] [Docker Image] [Command]
$ docker run -v [host path / volumne name]:[container path]  -it --rm [docker image] [command] 
$ # Example:
$ docker run -v rvm:/home/jenkins/.rvm -v jenkins_workspace:/home/jenkins/workspace -it --rm joshua5201/jenkins-slave-rails /bin/bash
```

## Prepare Docker Images
### Install RVM in Image (First run only)

1. Pull image: `docker pull joshua5201/jenkins-slave-rails`
2. Create volume for RVM: `docker volume create --name rvm`
3. Create volume for Workspace: `docker volume create --name jenkins_workspace`
4. Install RVM in docker: `docker run -v rvm:/home/jenkins/.rvm -it joshua5201/jenkins-slave-rails /bin/bash`
5. Inside docker: 
    - `su -l jenkins`
    - `curl -sSL https://get.rvm.io | bash -s stable`
6. You can just mount rvm volume whenever you add a docker image (~/.rvm must exists). 

### Available Images
1. `joshua5201/jenkins-slave-rails`: basic runtime 
2. `joshua5201/jenkins-slave-rails-pg`: with postgresql installed. postgres user: jenkins, no password.

## Jenkins Configuration
1. Follow the default steps and create first administrator user
2. Manage jenkins -> Manage plugins -> Available -> install docker rvm
3. Manage jenkins -> Configure system -> Add a new cloud (choose docker) ref: [https://wiki.jenkins-ci.org/display/JENKINS/Docker+Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Docker+Plugin)
    - set name, docker url (usually `unix:///var/run/docker.sock`)
<br><img src="Screenshots/cloud-config.png"><br>
    - Add docker template
    - Docker image: joshua5201/jenkins-slave-rails
    - Container settings -> Volumes: rvm:/home/jenkins/.rvm jenkins\_workspace:/home/jenkins/workspace
    - Remote File System Root: /home/jenkins
    - Labels: docker
    - Add Credentials -> username with password -> jenkins/jenkins
<br><img src="Screenshots/docker-config-1.png"><img src="Screenshots/docker-config-2.png"><br>
<br><img src="Screenshots/jenkins-credentials.png"><br>
4. When adding other images like jenkins-slave-rails-pg, just change the Docker image and Labels above. (e.g. docker-pg)

## Create Build Job
1. New Item -> Enter name -> Choose freestyle item
2. General -> Advanced -> Custom Workspace:  jenkins\_workspace:/home/jenkins/workspace
<br><img src="Screenshots/custom-workspace.png"><br>
2. Restrict where this project can be run: docker (or whatever labels you set for your docker image)
3. Source Code Management: git -> set repo url -> add credentials (ssh private key with username 'git')
4. Build Environment: Run the build in a RVM-managed environment -> choose your implementation (e.g. `2.3.0`)
5. Add build steps: Execute shell 
``` bash
gem install rubygems-bundler
sudo /etc/init.d/redis-server start # if you need Redis
bundle install
bundle exec rake db:test:prepare
bundle exec rspec 
```

## Create Deploy Job
1. To avoid SSH problems, please use the same key for GitHub repo deploy key and the key for SSH login to staging / production server.
2. Follow the same steps as build job.
3. At Build Environment -> Use secret text(s) or file(s) -> Varialble: DEPLOY\_KEY -> Upload a secret file (id\_rsa PRIVATE key)
<br><img src="Screenshots/deploy-key.png"><img src="Screenshots/add-key-file.png"><br>
4. Use the following shell script template:

``` bash
# Prepare bundler
gem install rubygems-bundler
bundle install

# Setting up ssh-agent for capistrano
eval `ssh-agent`
ssh-add $DEPLOY_KEY

# Deploy scripts here
bundle exec cap staging deploy

# Kill ssh-agent
kill $SSH_AGENT_PID
```

## Tips
1. You can create new job based on old ones.
2. If you want Jenkins to integrate with GitHub:
    - Go to [https://github.com/settings/tokens](https://github.com/settings/tokens) to generate your token
    - Manage Jenkins -> Configure System 
    - GitHub -> Add GitHub Server
    - Credentials: Secret Text -> Input your token here
<br><img src="Screenshots/github-hook.png"><br>
3. Project configuration tips: 
    - Build Triggers -> Build when a change is pushed to GitHub
    - Post-build Actions: Set status for GitHub commit (need to have token set) -> Status Result: One of default.
<br><img src="Screenshots/github-commit-status.png"><br>
4. If you want to bypass CI in commit message like [ci skip], install this plugin [ci-skip](https://wiki.jenkins-ci.org/display/JENKINS/Ci+Skip+Plugin)

## Troubleshooting
1. If any packages are needed to be installed, email: joshua841025@gmail.com or fork my Dockerfile.
