---
layout: post
title:  "Setting up a Docker environment"
date:   2023-01-06 15:55:50 +0100
category: Work
tags: SetupNotes CloudEngineering
---
<!readmore>

## Setting up a Docker environment, locally and on AWS
This is rather a collection of notes/learnings than a "post". I started with this documentation when I was in the preparation for my project about building text generators. Since I knew that I wanted to finally deploy (or train) on AWS, I thought it might be a good idea to use an environment which I could simply transfer to the cloud. Docker seemed to be such an environment, so I started setting it up locally and also experimented with transferring it to EC2 instances in AWS. 

## Developing code locally with VScode and a Docker container
- Prerequisites: 
    - Docker and WSL installed  
    - Have the `Remote - Containers` Extension installed in VScode  
    - A folder called `.devcontainer`. There you have your `Dockerfile` and a `devcontainer.json` file  
    - A `requirements_dev.txt` file in the folder of this README. This file is used in the `Dockerfile` to install the libraries with pip.  
- When you open this folder in VScode, you will be asked to reopen this folder in a container
- Alternatively: Ctrl+Shift+P and search for "open folder in container"
- If you changed something in the requirements or the dockerfile: Ctrl+Shift+P and search for "rebuild container without cache"
- A note on using the formatter "black": It will not work out of the box since the default path is set to /usr/local/...
    Go to Preferences/Settings and search for "black path". You have to set the path to "/root/.local/bin/black". I found out by manually trying to install black with pip under root, that the path is somewhere in /root/.local/lib. A quick search of /root/.local/bin showed that black is located there.


## Packaging a Docker image for AWS
- Each of the folders has its own Dockerfile. For example the folder `/backend/model_builder`
- cd into this folder and run:  
    ```bash
    docker build . -t model-builder; \  
    docker image save model-builder:latest -o model-builder.tar; \
    gzip model-builder.tar  
    ```
- Then move this tar file to a directory outside of this repo. Make sure you have the pem-file for the EC2 instance there as well (`model-builder.pem`):
    ```bash
    mv model-builder.tar.gz ~/Documents/Projects/resources/TextGenerators/; \
    cd ~/Documents/Projects/resources/TextGenerators/; \
    scp -i "/c/Users/.../nn-trainer.pem" model-builder.tar.gz ec2-user@ec2-x-xx-xxx-xx.eu-central-1.compute.amazonaws.com:/dltraining  
    ```


## Setting up a Docker image on an EC2 instance  
- **Note**: I haven't tested a container environment for the actual training of the text generators. It was unclear to me if GPUs could be accessed from the container and root-installing the environment simply seemed more efficient.  
- This section requires that you completed the sections "Setup an EBS volume" and "Packaging a docker image" 
- On the EC2 instance, run the following commands (setting access token environment variables are optional - I used them to access the Twitter API):  
    ```bash
    gunzip /tmp/model-builder.tar.gz; \
    docker image load -i /tmp/model-builder.tar; \
    docker run -d --restart always -e CONSUMER_KEY="..." -e CONSUMER_SECRET="..." -e ACCESS_TOKEN="..." -e ACCESS_TOKEN_SECRET="..." -it --mount type=bind,source="$(pwd)"/dltraining,target=/dltraining --name generator model-builder:latest /bin/bash
    ```
- the -d option runs the container in detached mode (in the background), the --restart always option assures that the bot will keep running if you disconnect from the SSH session or if the instance is restarted. the -it option (actually -i -t) is used for an interactive session and to allocate a tty for the container process.
- The last part (/bin/bash) is necessary in order to have a command line interface once you are inside of the container.
- With --mount you "bind mount" the folder dltraining, such that you can always copy additional files into the container
- To update the container you need to stop and remove it:
    ```bash
    docker stop <container-id/name>
    docker rm <container-id/name>
    ```


## Connecting VScode to the docker container on EC2
- Using these tutorials:  
    - [VS Code remote overview][vscode-remote]  
    - [VS Code SSH tutorial][vscode-ssh]  
- Install the "Remote Development" extension pack.
- In VSCode you need to specify the config file in Ctrl+Shift+P: "Remote-SSH open SSH configuration file"
    `C:\Users\...\AppData\Local\Programs\Git\etc\ssh\ssh_config`
In the config file, add this block:
    ```bash
    Host aws-ec2
        HostName xx.xx.xxx.xxx
        User ubuntu
        IdentityFile C:\Users\...\model-builder.pem
    ```
    Make sure that you comment out the default blocks in the config files that were added by git. VScode will use OpenSSH_for_Windows, and OpenSSH doesn't know keywords like "PubkeyAcceptedAlgorithms" (in Host ssh.dev.azure.com)       
- Use the status icon in the lower left (><) or Ctrl+Shift+P: "Remote-SSH: Connect to Host (aws-ec2 can be selected)
- Once you are connected you can use Ctrl+Shift+P: "Remote-Containers: Attach to running containers" and you will be able to select the container that you started on the instance.


[vscode-remote]: https://code.visualstudio.com/docs/remote/remote-overview
[vscode-ssh]: https://code.visualstudio.com/docs/remote/ssh-tutorial
