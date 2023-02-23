---
layout: post
title:  "Building a text generator on AWS"
date:   2023-02-16 20:55:50 +0100
tags: CloudServices MachineLearning
---
# Transformers and the Cloud
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


# The recipe
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

## Collecting data from mtgjson.com project
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

## The simplest model
A Markov chain is one of the simplest approaches to text generation. In simple terms, a Markov chain looks at the current word (or group of letters) and chooses the next word (or group of letters) with a certain probability. The probabilities for the next word in the sequence are calculated from a larger text corpus. Or we say, the model is "fitted" on a training dataset (well, not with gradient descent but you know what I mean).

![Markov Chain](assets/images/experimenting-with-text-generators/markovchain.jpg)
By Joxemai4 - Own work, CC BY-SA 3.0, https://commons.wikimedia.org/w/index.php?curid=10284158

### How it works
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

### Markov chain outputs
Here is a selection of sentences generated by the Markov chain model:  
  
- Its low hum reminds me of angelic saviors' wings. They see its wings. The sky requires it. But it is right to make sure it was right.
- All da bugs here got wings!   (I love this one)
- It may now be time for more chaotic violence.
- Dreams are the path.
- We bury our past to soil - until I've gnawed their natural magical device.
- Supporting sun and sunny places. How would I kill the sky.  (I can only agree with this one - nice!)

Theses examples actually can make sense to a reader but they are extremely rare: to find them I searched 230 lines of output text.  
I've seen this Markov chain model perform better on producing song lyrics. I could imagine that lyrics work better because of a more concise "sentence" structure and use of words.   

## The next evolutionary step: LSTM RNNs 
The next models I tried were the long / short term memory recurrent neural networks (LSTM RNNs). These can still be trained locally (and I did for the prototyping) but only if you use a small number of epochs. A thorough training requires something more powerful than my laptop. Let's first look at the core parts of such a recurrent neural network in code.  

### How it works
As mentioned above, if you check out [Create your first LSTM][create-first-lstm], the article also links to another one which summarize how LSTM RNNs work. All credit (especially for the code) goes to these authors. But I will do my best to give my own brief description here.  

A recurrent neural network is a network that can be used to generate a series of outputs, where each new output is influenced by the preceding inputs. They have an internal "state" which keeps changing with the sequence of inputs and acts just like a short-term memory. However, there is an issue with longer sequences: The internal state is not able to recall what happened at the beginning of a long sequence due to the well known vanishing gradient problem. This is where Long Short-Term Memory RNNs come into play. These were specifically designed to handle long-term dependencies. In an LSTM RNN, there is a simple RNN cell, a dedicated long-term memory cell, and three gates: an input gate, a forget gate, and an output gate. These control the flow of information into and out of each "memory" cell.  

![LSTM](assets/images/experimenting-with-text-generators/lstm.png)
By Guillaume Chevalier - File:The_LSTM_Cell.svg, CC BY-SA 4.0, https://commons.wikimedia.org/w/index.php?curid=109362147  

The forget gate determines how much of the previous hidden state should be retained. The input gate determines whether the memory cell is updated and how much new information should be added to the memory cell state. A new cell state is created based on the output of the input and forget gates. Finally the output gate determines what the next hidden state should be from the newly calculated memory cell state, the previous hidden state, and the input data.

One benefit of LSTM RNNs over normal RNNs is that they can adjust the weights of these gates during the training and find out on their own which long-term dependencies to look for in the data.

In my experiments, I saw that usually, a single LSTM layer is not better than a Markov chain model. To achieve a better performance, I stacked 3 bidirectional LSTM layers. What bidirectional layers are is explained [here][keras-lstm-guide], but in brief, they allow the network to not only process a sequence from start to end but also backwards. This gives the model more awareness of the "entire" context arount a word (i.e. considering words before and after a certain word), where a regular LSTM would ignore everything that follows a given word.  This allows the network to capture both short-term and long-term dependencies in the data. The outputs from the LSTMs at each time step are typically fed into a final output layer that produces the final prediction.

## Switch to the cloud environment
To prevent this article from growing too big, I have moved the whole part of setting up the AWS environment to [this post]({% post_url 2023-02-15-first-setup-in-aws-and-using-ec2-for-machine-learning-tasks %}). There you'll find details on things like creating the necessary IAM roles, moving your data to EBS volumes, or creating templates to launch your training jobs. But let's talk about more interesting stuff.

### LSTM RNN outputs
The LSTM produces sentences such as:

- The Conch will kill those whom once they loved.
- His drafts are barely aware and reason. I regret feeding grounds mortals who have toucted the entrance 
- When the spirit is just the best answers will endure.
- For the lost time, a flower mage can find it had heard the recognize it, as never mistake it.
- It reflects the structure that means no cave far are many way abomination of a defense.

## Transformers
...

### How it works
...

### Transformer outputs
Here is a selection of sentences generated by the Markov chain model:  

- Two glowing red eyes and the rustling of the stones means that some people believe that they are the guardian spirits of the forest.
- You steal it, I find it. You then buy it back again, and I'm back here in hell.
- The one you see is just a diversion. There is no way around it. When you hit it, you break it. The trick is to drive it away from you.
- The mycosynth created the first aliens.


[karpathy-rnns]: http://karpathy.github.io/2015/05/21/rnn-effectiveness/
[deep-drumpf]: https://www.inverse.com/article/12418-donald-trump-artificial-intelligence-neural-network  
[twitter-bot-with-tweepy]: https://realpython.com/twitter-bot-python-tweepy/  
[lyrics-with-markov]: https://realpython.com/lyricize-a-flask-app-to-create-lyrics-using-markov-chains/  
[create-first-lstm]: https://towardsai.net/p/deep-learning/create-your-first-text-generator-with-lstm-in-few-minutes  
[tuning-gpt2]: https://towardsdatascience.com/how-to-fine-tune-gpt-2-for-text-generation-ae2ea53bc272  
[keras-lstm-guide]: https://keras.io/guides/working_with_rnns/
