---
layout: post
title:  "Building a text generator on AWS"
date:   2023-01-16 20:55:50 +0100
tags: CloudServices MachineLearning
---
# Introduction and motivation
I started this project because I was curious about a sub-field of machine learning which is currently more hyped than quantum computing. Companies spend millions on this technology just because it is so versatile and applicable in our everyday lives. I'm talking about *language models*.  
Without having any prior experience in this domain, I saw two things that I was curious about:  
- Lately the academic community as well as the industry switched from recurrent neural networks to so called *transformer* architectures. It seemed to be fairly straightforward to obtain one of these models pre-trained on a large text dataset, and then fine-tune it, such that the model is able to produce text in the same style as your training data (e.g. "write like Shakespeare"). So this is point number one that sparked my interest.  
- Second, I saw some opportunities to explore a new topic which was very relevant for my daily work: *Cloud services*. Cloud services like the ones you get from Amazon Web Services (AWS), Microsoft Azure, or Google Cloud Platform come handy when you want to train and run these language models. In the cloud, it is simple to rent hardware you would never buy privately. I think for the entire project I spent something in the order of 30 €, which is little given the machines that I used. If you just want to play around, I believe that Google Colab or Kaggle Notebooks are your best options - you can use a GPU or TPU for quite some time without charge. My company has recently moved to the AWS ecosystem, so that simplifies the choice for me.  
  
That's basically why I wanted to come up with a project to bring these things together: Learning a little about the latest fad in machine learning and getting more familiar with state-of-the-art cloud services.
  
  
If you are interested in some articles on the topic, these are the ones that inspired me to do this project:  
- [The Unreasonable Effectiveness of Recurrent Neural Networks][karpathy-rnns]
- "Deep Faking" Political Twitter using Transfer learning and GPT-2; 2019  
- [Deep Drumpf][deep-drumpf]
- [Building a lyrics generator with markov chains][lyrics-with-markov]
- [Create your first LSTM][create-first-lstm]
- [Fine tuning GPT-2][tuning-gpt2]


# Project definition
What are the ingredients of this project? Some text data, a language model, and an account in AWS. Playing with the language models was a big part, and here I tried to follow the tracks of history, to see where the current state was coming from:
- The simplest approach to try was a *Markov chain model*. It's just 80 lines of code, which you can still implement without fancy packages, plus, you'll immediately understand what every line is doing.
- Going in big leaps up the evolutionary ladder of models, I wanted to try out *long / short term memory recurrent neural networks* (LSTM RNNs) - if you check out [Create your first LSTM][create-first-lstm], there are nice links which summarize how RNNs / LSTM RNNs work.
- And finally, there's the (current) masterclass: *transformers*. It's beyond my time constraints to implement a transformer myself, but with Hugging Face's transformers library it's only a matter of using the API in the right way.  
  
The second big package was getting set up in the AWS environment. Initially I thought about setting up all environments for training and deployment in docker (well, you know the hype). But I came across two reasons why it might not be a good idea for a simple project like mine, or maybe even your typical data science project:
- First, complexity: Docker adds an additional layer of complexity to the training and deployment process, especially for a project where you have limited time resources and the focus is to build a prototype.
- From my experience, added complexity (in the cloud) goes hand in hand with some overhead, which you should only accept if absolutely neccessary. Basically with my small dataset and me being the only user, ends don't justify the means.


# Let's go step by step

## Preparation and local testing
At the very beginning of this project, I immediately thought of Twitter as a potential source for text data. Wikipedia was another option but Twitter had the appeal to contain more or less unfiltered quotes (so how people actually think and talk). Well, initially I did go the Twitter route but this is maybe a story for a different post since it would make this one unneccessarily long. In this post, I'd rather use a shortcut and focus on more interesting topics: the machine learning- and cloud services part. My shortcut is a dataset of *Magic: The Gathering* cards, where you can easily extract all the so called "flavor texts". These are snippets of text which are like quotes from a "Lord of the rings" book. At least to me, that was an interesting dataset, since I started to play MTG on the side at that time. But feel free to replace it with whatever other text corpus you are interested in (maybe Shakespeare or Dostoyevsky?). Doing all of this should be about the fun of it, so if you want to try it for yourself, pick something that makes it fun. 

### Collecting data from mtgjson.com project
In case you want to follow my tracks, go to mtgjson.com and download the AllPrintings.json from the *Downloads* section. With a simple Python script you can extract the flavor text snippets and save them to a csv file. Just a quick note: I will try to reduce code snippets to core functionality and leave out boilerplate stuff.  
```python
# import [...]

with open(os.path.join(source_folder_string, "AllPrintings.json")) as f:
    db = json.load(f)

flavorText_list = []

for s in db["data"].keys():

    for c in db["data"][s]["cards"]:
        if "flavorText" not in c.keys():
            continue

        if c["language"] == "English":

            text = c["flavorText"]
            if "—" in text:
                text = text[: text.index("—")]

            flavorText_list.append(text)

df = pd.DataFrame(flavorText_list, columns=["Text"])
df.drop_duplicates(subset=["Text"], inplace=True)
df.to_csv(output_path_string)
```

### The simplest model
A Markov chain is one of the simplest approaches to text generation. In simple terms, a Markov chain looks at the current word (or group of letters) and chooses the next word (or group of letters) with a certain probability. The probabilities for the next word in the sequence are calculated from a larger text corpus. Or in ML speak, the model is "fitted" on a training dataset (well, not with gradient descent but you know what I mean).

Usually we speak of so called "*n*-gram" models, where *n* is the number of words that the model considers to predict the next word. Based on this mechanic, the generated text is very similar in style to the input text. The core functionality can be discussed with the following two functions (taken from [Building a lyrics generator with Markov chains][lyrics-with-markov])  
```python
# import [...]

def generateModel(text, order, model=None):
    """
    :param text:    A string of text which the model is generated from.
    :param order:   An integer representing the length of the fragment.
    :param model:   An optional parameter representing the language model. It is initialized as an empty dictionary {} if not provided.

    :return:         A language model in the form of a dictionary 
    """

    if model == None:
        model = {}
    
    for i in range(0, len(text) - order):
        fragment = text[i:i+order]
        next_letter = text[i+order]
        if fragment not in model:
            model[fragment] = {}
        if next_letter not in model[fragment]:
            model[fragment][next_letter] = 1
        else:
            model[fragment][next_letter] += 1
    
    return model

def getNextCharacter(model, fragment):
    letters = []
    for letter in model[fragment].keys():
        for times in range(0, model[fragment][letter]):
            letters.append(letter)
    
    return random.choice(letters)
```
The `generateModel` function generates a simple language model in the form of a dictionary. In this dictionary, each key is a fragment of text with length equal to `order` parameter. The key's value is another dictionary, where the keys are "next letters" observed to follow a given fragment in the text and the values are the number of times those letters appear after the given fragment. The function then scans the text by sliding a window of length `order` over the text and counts the number of occurrences of "next letters" following each fragment. The resulting dictionary is our first simple "language model".
  
Finally, with the `getNextCharacter` function we can simply provide a stream of fragments, and in a probabilistic way predict the "next letter" for each fragment. To do so, the function looks into the dictionary and pulls out all letters observed to follow a certain text fragment. Then these letters are appended to the list `letters` as many times as they were observed in the original dataset to follow the respective fragment. From the resulting list we can choose a letter at random and automatically get a good match for our given fragment. Not only has it definitely been observed after the given fragment, but it will also have a good chance of beeing one of the most frequent letters to follow our fragment.  

### First results
Here is a selection of sentences from the Markov chain model:  
  

## Setup in AWS training a first model 

### Step by step setup of your AWS environment
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
    ```bash
    sudo apt-get update; \
    sudo apt install docker.io; \
    sudo adduser ubuntu docker; \
    exit
    ```

### Setup an EBS volume
- Data transfers will be managed with a general purpose instance. You can create it with the CLI as follows:
    ```bash
    aws ec2 run-instances \
        --image-id ami-09439f09c55136ecf \
        --count 1 \
        --instance-type t2.micro \
        --key-name nn-trainer \
        --query "Instances[0].InstanceId"
    ```
- ami-09439f09c55136ecf is an "Amazon Linux 2" image; the default user is "ec2-user"
- Next, create an EBS volume for your datasets and checkpoints:
    ```bash
    aws ec2 create-volume \
        --size 4 \
        --region eu-central-1 \
        --availability-zone eu-central-1b \
        --volume-type gp2 \
        --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=DL-datasets-checkpoints}]' 
    ```
- Save the volume-id returned by the previous command then attach the volume:
    ```bash
    aws ec2 attach-volume \
        --volume-id vol-<your_volume_id> \
        --instance-id i-<your_instance_id> \
        --device '//dev\sdf'
    ```
    Depending on the operating system it has trouble with the --device parameter. On Windows in the Git bash, I needed '//dev\sdf'.
- Now you have to create a directory on the EC2 instance and mount it. You cannot directly ssh to the ec2 instance. In EC2 / Instances / your instance / Security / click the security group / you have to delete the old inbound rule in the security group and add a new rule: to allow your IP on port 22.
- If you are attaching a blank EBS volume, use:
    ```bash
    sudo mkdir /dltraining; sudo mkfs -t xfs /dev/xvdf; sudo mount /dev/xvdf /dltraining; sudo chown -R ec2-user: /dltraining/; cd /dltraining; mkdir datasets; mkdir checkpoints
    ```
- If you did this step before, the instance will already have the folder /dltraining and the EBS volume will already contain data from a previous run, you only need to mount it:
    `sudo mount /dev/xvdf /dltraining; sudo chown -R ec2-user: /dltraining/;`
- Prepare files on the EBS volume prior to mounting it for training:
    - Copy over the necessary files:
    ```bash
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
    ```bash
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
    ```bash
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
    ```bash
    aws iam create-role \
        --role-name DL-Training \
        --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Sid":"","Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
    ```
- Next, create and attach a policy that grants the instance the following permissions:
    Describe, create, attach and delete volumes
    Create snapshots from volumes
    Describe spot instances
    Cancel spot fleet requests and terminate instances
    ```bash
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
    ```bash
    aws iam create-role \
        --role-name DL-Training-Spot-Fleet-Role \
        --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Sid":"","Effect":"Allow","Principal":{"Service":"spotfleet.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

    aws iam attach-role-policy \
         --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2SpotFleetTaggingRole --role-name DL-Training-Spot-Fleet-Role
    ```
- You have to encode the user_data_script.sh with base64: 
    ```bash
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

