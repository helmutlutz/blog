---
layout: post
title:  "Collecting a dataset from Twitter"
date:   2023-01-16 20:55:50 +0100
tags: CloudServices MachineLearning
---
# Intro
At the very beginning of my [Building a text generator on AWS]({% post_url 2023-01-16-experimenting-with-text-generators %}) project, I immediately thought of Twitter as a potential source for text data. Wikipedia was another option but Twitter had the appeal to contain more or less unfiltered quotes (so how people would think and talk). Specifically I was interested in using such a datasets to fine-tune the models to imitate the writing style of some popular personalities (think of [Deep Drumpf][deep-drumpf] example).

# Outline


# Step by step

## Collecting data with the Twitter API
To collect data from Twitter, you will need to create a developer account on the Twitter Developer website and create a Twitter App. Once you have created the app, you will be given a set of API keys and tokens (consumer key, consumer secret, access token, and access token secret) that you can use to authenticate your requests to the API.
  
Once you have the API keys and tokens, you can use a programming language such as Python to interact with the API. There are several libraries available that make it easy to work with the Twitter API, such as the "tweepy" library for Python.
  
You can use the library to connect to the twitter API, authenticate your requests using the keys and tokens, and then make requests to the API to collect data. You can also use the library to filter the data by different parameters.
  
Here is an example of how you can use the tweepy library to collect tweets containing a certain hashtag:

```python
import tweepy

# Authenticate using your API keys and tokens
auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
auth.set_access_token(access_token, access_token_secret)
api = tweepy.API(auth)

# Collect tweets containing the hashtag "example"
hashtag = "#example"
tweets = api.search(q=hashtag)

# Print the text of the tweets
for tweet in tweets:
    print(tweet.text)
```

You can also use the API endpoints to get specific data such as user's tweets, follower, following etc.
You can refer to the twitter API documentation for more information on how to use the API and the different endpoints available.

- Find your consumer key, consumer secret, access token, and access token secret.
- In the VScode terminal, execute:
  `export CONSUMER_KEY="...."` (for all four keys)
- Execute the command:
  `cd /backend/data_collector; python profile_scraper.py -p <some-twitter-handle> -f`