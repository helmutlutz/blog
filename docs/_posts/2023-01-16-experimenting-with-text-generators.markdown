---
layout: post
title:  "Building a text generator on AWS"
date:   2023-01-16 20:55:50 +0100
tags: CloudServices MachineLearning
---
# Intro to the project
I started this project because I was curious about a technique, which, despite being a niche (from a theoretical perspective), is totally hyped right now. People even build huge machine learning systems just because they are so versatile and applicable in our everyday lives. The ML systems I'm talking about are called *language models*.  
In this particular domain I saw multiple things that I was curious about: Lately the research and developer community switched from recurrent neural networks to *transformer* architectures. It is possible to train these models on large, unspecific datasets, and then fine tune them to a particular style (or author), such that your model is able to produce text in the same style.  
Also from an operational point of view, I saw some opportunities to explore a new topic which was relevant for my daily work: *cloud services*. Cloud services like the ones you get from Amazon Web Services (AWS), Microsoft Azure, or Google Cloud Platform come handy when you want to train and run these language models. In the cloud you are 'free' to rent hardware you wouldn't normally buy for your home PC. I think for the entire project I spent something in the order of 30 €. If you just want to play around, I believe that Google Colab or Kaggle Notebooks are your best options - you can use a GPU or TPU for quite some time without charge. However, my company has moved to the AWS ecosystem, so that relieves me of picking a provider myself.
So that's basically why I wanted to come up with a project to bring these two things together: Learning a little about new frontiers in machine learning and getting more familiar with our tech stack.
  
If you are interested in some articles on the topic, these are the ones that inspired me to do this project:  
- [The Unreasonable Effectiveness of Recurrent Neural Networks][karpathy-rnns]
- "Deep Faking" Political Twitter using Transfer learning and GPT-2; 2019  
- [Deep Drumpf][deep-drumpf]
- [Building a lyrics generator with markov chains][lyrics-with-markov]
- [Create your first LSTM][create-first-lstm]
- [Fine tuning GPT-2][tuning-gpt2]

## The plan
So how to best put together a project that has lots of learning potential? 

## Preparation and local testing

### Collecting data
- Find your consumer key, consumer secret, access token, and access token secret.
- In the VScode terminal, execute:
  `export CONSUMER_KEY="...."` (for all four keys)
- Execute the command:
  `cd /backend/data_collector; python profile_scraper.py -p <some-twitter-handle> -f`


### Training a first model
- You can use the script `/backend/model_builder/rnn_builder.py` as a standalone solution to train and generate some text. To run it, you need to be in the root directory (where the folders `/backend` and `/lib` are located)
    ```
    # Train the first model with '-t':
    /usr/local/bin/python TextGenerators/backend/model_builder/rnn_builder.py -t
    or
    /usr/local/bin/python TextGenerators/backend/model_builder/gpt2_tuner.py -t -p ClimateLeader
    
    # Can also be used for inference by providing an initial seed string with '-s':
    /usr/local/bin/python TextGenerators/backend/model_builder/rnn_builder.py -s 'They ' 
    ```


## Setup in AWS and deployment of the training

### Setup of AWS
- Create an AWS account
- Follow this tutorial to create an administrator IAM user and user group (console):
    https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html
    When the Admin user is created and added to the group of Administrators, you will see the Access Key ID and the Secret Access Key. Save these, they will be needed to set up the CLI 
- The following two steps are needed if you want to use EBS/ECS:
- If not already installed, install the AWS CLI (command line interface)
- Configure the aws cli by going into the git-bash:
    `aws configure`
    Enter the Access Key ID, the Secret Access Key (you got these when creating the Administrator User in "Setup of AWS") and the default region (eu-central-1), output can be set to json.

### Setup of an EC2 instance for hosting
- Follow this tutorial to set up a micro-instance for hosting: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html
- It is important to create and store a Key-Pair (`.pem` file). You'll need this to ssh to the instance.
- Edit the inbound rules in your security group to allow SSH on port 22 from **your current IP**.
- Now ssh to the instance: AWS suggests the command when you right-click the instance in the dashboard and select "Connect"
    ` ssh -i "/c/Users/.../nn-trainer.pem" ubuntu@ec2-xx-xx-xxx-xxx.eu-central-1.compute.amazonaws.com`  # With Ubuntu AMI
    or
    `ssh -i "/c/Users/.../nn-trainer.pem" ec2-user@ec2-x-xx-xxx-xx.eu-central-1.compute.amazonaws.com`  # With Amazon Linux 2 AMI
- Optional: Install Docker with the following commands:
    ```
    sudo apt-get update; \
    sudo apt install docker.io; \
    sudo adduser ubuntu docker; \
    exit
    ```

### Setup an EBS volume
- Data transfers will be managed with a general purpose instance. You can create it with the CLI as follows:
    ```
    aws ec2 run-instances \
        --image-id ami-09439f09c55136ecf \
        --count 1 \
        --instance-type t2.micro \
        --key-name nn-trainer \
        --query "Instances[0].InstanceId"
    ```
- ami-09439f09c55136ecf is an "Amazon Linux 2" image; the default user is "ec2-user"
- Next, create an EBS volume for your datasets and checkpoints:
    ```
    aws ec2 create-volume \
        --size 4 \
        --region eu-central-1 \
        --availability-zone eu-central-1b \
        --volume-type gp2 \
        --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=DL-datasets-checkpoints}]' 
    ```
- Save the volume-id returned by the previous command then attach the volume:
    ```
    aws ec2 attach-volume \
        --volume-id vol-<your_volume_id> \
        --instance-id i-<your_instance_id> \
        --device '//dev\sdf'
    ```
    Depending on the operating system it has trouble with the --device parameter. On Windows in the Git bash, I needed '//dev\sdf'.
- Now you have to create a directory on the EC2 instance and mount it. You cannot directly ssh to the ec2 instance. In EC2 / Instances / your instance / Security / click the security group / you have to delete the old inbound rule in the security group and add a new rule: to allow your IP on port 22.
- If you are attaching a blank EBS volume, use:
    ```
    sudo mkdir /dltraining; sudo mkfs -t xfs /dev/xvdf; sudo mount /dev/xvdf /dltraining; sudo chown -R ec2-user: /dltraining/; cd /dltraining; mkdir datasets; mkdir checkpoints
    ```
- If you did this step before, the instance will already have the folder /dltraining and the EBS volume will already contain data from a previous run, you only need to mount it:
    `sudo mount /dev/xvdf /dltraining; sudo chown -R ec2-user: /dltraining/;`
- Prepare files on the EBS volume prior to mounting it for training:
    - Copy over the necessary files:
    ```
    cd /c/Users/.../TextGenerators/backend/model_builder; \
    scp -r -i "/c/Users/.../nn-trainer.pem" \
        requirements_rnn.txt rnn_builder.py datasets \
        ec2-user@ec2-3-73-123-91.eu-central-1.compute.amazonaws.com:/dltraining
    ```
- **Important**: Unmount and detach the EBS volume after you have copied datasets and scripts into it.
    - `sudo umount -d /dev/xvdf`
    - The volume can be detached in the EC2 console.
    - Useful to get an overview of mounted volumes: `findmnt`  
- You can terminate the instance with this command:
    ```
    aws ec2 terminate-instances \
        --instance-ids i-<your_instance_id> \
        --output text
    ```
- **Shortcut**: Once the EBS volume exists, you can use the launch template `user_data_script_file_management.sh`, which automates the process of starting an instance and mounting the volume for you. Just edit the security group's inbound rules and the instance is ready for ssh/scp access.  

### Setup EC2 instance for training with launch template
- Reference: Tutorial for automatically attaching a persistent secondary EBS volume to a new EC2 Linux Spot Instance at boot
    https://aws.amazon.com/premiumsupport/knowledge-center/ec2-linux-spot-instance-attach-ebs-volume/?nc1=h_ls

- The first step will be to set up a dedicated EBS (elastic block store) volume to store data and logs. This is described in the section "Setup an EBS volume"  
- **Important**: Unmount and detach the EBS volume after you have copied datasets and scripts into it.
    - `sudo umount -d /dev/xvdf`
    - The volume can be detached in the EC2 console.
    - Useful to get an overview of mounted volumes: `findmnt`  
- POC: Manually start an instance and try out the steps below and from the user_data_script_train_template.sh (mounting and unmounting, executing the python script, etc.)
- Then, start with setting up the template script by navigating to EC2 / Instances / Launch Templates
- Select g5.xlarge or g4dn.xlarge as the instance type  
- Select the Deep Learning AMI GPU ...
    ... TensorFlow (for rnn_builder): ami-00952eca414e66cbe
    ... PyTorch (for gpt2-tuner): ami-084f03f64414b042f
- Note: Could also work but I haven't tested the Deep Learning Base AMI yet: ami-0acb218a9a0302218 (it has only GPU drivers and Docker, no python packages)
- In "Advanced details", where you should add the IAM role, there was only a selection field for an IAM Profile
- There was no description for how to create an IAM profile for this step, but I could use the solution described here: https://aws.amazon.com/premiumsupport/knowledge-center/iam-role-not-in-list/
    ```
    aws iam create-instance-profile --instance-profile-name DL-Training
    aws iam add-role-to-instance-profile --role-name DL-Training --instance-profile-name DL-Training
    ```
    (I reused the IAM role 'DL-Training' that I created in the section "Setup an automatic spot instance fleet request for training")
- If you want the instance to self-terminate (with the last command in user_data_script_train_template.sh), you have to select "Terminate" in the dropdown for "Shutdown behavior" 
- Paste your user_data_script_train_template.sh (in this repo next to this README) in the respective section.
- To verify proper setup of the GPU, use 
    `python3 -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"`
- Optional, not tested yet: The use of Docker 
    - Transfer the docker image via the t2.micro instance into the EBS volume
    - Edit the user data script so that it spins up the docker container

### Track training progress on EC2
- This goes in particular for the gpt2_tuner.py script, which writes a train.log file with the loss
- Get Gnuplot on the EC2 instance with `sudo yum install gnuplot`
- copy the file /dltraining/checkpoints/train.log to your current working directory. (Otherwise the file will be blocked during training)
- Enter gnuplot
    `gnuplot> set datafile separator ","; set terminal dumb; plot "train.log" using 1:2 title 'Training Loss'`
  
- This is what the result looks like:  
    4 ++--------+---------+---------+--------+---------+---------+--------++  
      +         +         +         +        +        Training Loss   A    +  
  3.5 +AA                                                                 ++  
      |  A                                                                 |  
      |   A                                                                |  
    3 ++                                                                  ++  
      |                                                                    |  
  2.5 ++    A                                                             ++  
      |      A                                                             |  
    2 ++   A                                                              ++  
      |       A                                                            |  
      |        A                                                           |  
  1.5 ++                                                                  ++  
      |         A                                                          |  
    1 ++                                                                  ++  
      |          AA                    A                          AA  A    |  
      |              AAA         AA AAA  AAA AA  AA AA  A     A AA  AA     |  
  0.5 ++           AA   A AAAAA A  A    A      AA  A  AA AAAAA A       A  ++  
      +         +        A+    A    +       A+         +         +         +  
    0 ++--------+---------+---------+--------+---------+---------+--------++  
      0         10        20        30       40        50        60        70  

### Setup EC2 instance for inference with launch template
At first, I used the same instance as for the training. But I noticed that there was no GPU utilization during training (> nvidia-smi).  
The template was adjusted to use c5n.4xlarge (alternatively one of the c5a instances)

### Setup an automatic spot instance fleet request for training:
- Tutorial for using Amazon EC2 Spot Instances:  
    https://aws.amazon.com/de/blogs/machine-learning/train-deep-learning-models-on-gpus-using-amazon-ec2-spot-instances/

- The first step will be to set up a dedicated EBS (elastic block store) volume to store data and logs. This is described in the section "Setup an EBS volume"
- During training, I want the spot instance to have access to my datasets and checkpoints in the EBS volume I created in step 1. However, only volumes in the same Availability Zone as the instances can be attached to it. If the volume and the instance are in different Availability Zones, a new volume needs to be created using a snapshot of the volume stored in Amazon S3. 
    ```
    aws iam create-role \
        --role-name DL-Training \
        --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Sid":"","Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
    ```
- Next, create and attach a policy that grants the instance the following permissions:
    Describe, create, attach and delete volumes
    Create snapshots from volumes
    Describe spot instances
    Cancel spot fleet requests and terminate instances
    ```
    aws iam create-policy \
        --policy-name ec2-permissions-dl-training  \
        --policy-document file://ec2-permissions-dl-training.json
 
    aws iam attach-role-policy \
        --policy-arn arn:aws:iam::<account_id>:policy/ec2-permissions-dl-training \
        --role-name DL-Training
    ```
- The script checks with the volume and the instance are in the same Availability Zone. If they are in different Availability Zones, it first creates a point-in-time snapshot of the volume in Amazon S3. Once the snapshot is created, it deletes the volume and creates a new volume from the snapshot in the instance’s Availability Zone.
- Next, adapt the spot fleet configuration file (spot_fleet_config.json) that includes target capacity (e.g. 1 instance), launch specifications for the instance, and the maximum price that you are willing to pay. Simple 1-GPU instances are g5.xlarge or p2.xlarge (a little more expensive). Use the correct key-pair-name and a security group that allows you to ssh into the instance.
- To use the spot fleet Request, create an IAM fleet role by running the following commands:
    ```
    aws iam create-role \
        --role-name DL-Training-Spot-Fleet-Role \
        --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Sid":"","Effect":"Allow","Principal":{"Service":"spotfleet.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

    aws iam attach-role-policy \
         --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2SpotFleetTaggingRole --role-name DL-Training-Spot-Fleet-Role
    ```
- You have to encode the user_data_script.sh with base64: 
    ```
    USER_DATA=`base64 user_data_script.sh -w0`
    sed -i '' "s|base64_encoded_bash_script|$USER_DATA|g" spot_fleet_config.json 
    ```
- Submit the Spot fleet request `aws ec2 request-spot-fleet --spot-fleet-request-config file://spot_fleet_config.json`
- Cancel the running spot fleet request by issuing `aws ec2 cancel-spot-fleet-requests`


[karpathy-rnns]: http://karpathy.github.io/2015/05/21/rnn-effectiveness/
[deep-drumpf]: https://www.inverse.com/article/12418-donald-trump-artificial-intelligence-neural-network  
[twitter-bot-with-tweepy]: https://realpython.com/twitter-bot-python-tweepy/  
[lyrics-with-markov]: https://realpython.com/lyricize-a-flask-app-to-create-lyrics-using-markov-chains/  
[create-first-lstm]: https://towardsai.net/p/deep-learning/create-your-first-text-generator-with-lstm-in-few-minutes  
[tuning-gpt2]: https://towardsdatascience.com/how-to-fine-tune-gpt-2-for-text-generation-ae2ea53bc272  

