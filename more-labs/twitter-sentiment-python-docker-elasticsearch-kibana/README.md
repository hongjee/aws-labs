# Twitter Sentiment Analysis – Python, Docker, Elasticsearch, Kibana

We’ll connect to the Twitter Streaming API, gather tweets (based on a keyword), calculate the sentiment of each tweet, and build a real-time dashboard using the [Elasticsearch and Kibana](https://www.elastic.co/downloads/) to visualize the results.

> Tools: Docker, Tweepy v3.10.0, [TextBlob]((http://textblob.readthedocs.org/en/dev/) v0.16.0, Elasticsearch v7.10.2, Kibana v7.10.2

[TOC]

## Docker Environment

Follow the [official Docker documentation](https://docs.docker.com/desktop/) to install Docker and run 'docker version' to test the Docker installation. 

Create a directory to house the project, then create a Dockerfile
```
# start with a base image
FROM ubuntu:18.04

MAINTAINER Hong Jee <hongjee@yahoo.com>

# initial update and install wget
RUN apt-get update -q && apt-get install -yq wget

# create a new user elasticuser
RUN groupadd --gid 1001 elasticsearch && \
    useradd --system -create-home --home-dir /home/elasticsearch --shell /bin/bash --gid elasticsearch --groups sudo --uid 1001 elasticsearch
	
# install elasticsearch
RUN [ -d /opt ] || mkdir /opt
RUN cd /opt && \
    wget -nv https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.2-linux-x86_64.tar.gz && \
    tar zxf elasticsearch-7.10.2-linux-x86_64.tar.gz && \
    rm -f elasticsearch-7.10.2-linux-x86_64.tar.gz && \
    mv elasticsearch-7.10.2 elasticsearch && \
	echo 'transport.host: 127.0.0.1' >> /opt/elasticsearch/config/elasticsearch.yml && \
	echo 'http.host: 0.0.0.0' >> /opt/elasticsearch/config/elasticsearch.yml && \
	chown -R elasticsearch elasticsearch && \
# install kibana
    wget -nv https://artifacts.elastic.co/downloads/kibana/kibana-7.10.2-linux-x86_64.tar.gz && \
    tar zxf kibana-7.10.2-linux-x86_64.tar.gz && \
    rm -f kibana-7.10.2-linux-x86_64.tar.gz && \
    mv kibana-7.10.2-linux-x86_64 kibana && \
	chown -R elasticsearch kibana && \
# create entrypoint.sh 
	echo '#!/bin/bash' >> entrypoint.sh && \
	echo '/opt/elasticsearch/bin/elasticsearch -d' >> entrypoint.sh && \
	echo '/opt/kibana/bin/kibana -H 0.0.0.0 -p 5601' >> entrypoint.sh && \
	chmod a+x entrypoint.sh && \
	chown elasticsearch entrypoint.sh
	
USER elasticsearch
	
# expose ports
EXPOSE 9200 5601

# start elasticsearch
ENTRYPOINT ["/opt/entrypoint.sh"]
```

Build the image:
```bash
docker build -t hongjee/elasticsearch-kibana .
```

Launch Docker contaner to run Elasticsearch and Kibana
```bash
docker run  -d --name es7 -p 5601:5601 -p 9200:9200 hongjee/elasticsearch-kibana
```
or
```bash
docker run --rm --name es7 -p 5601:5601 -p 9200:9200 hongjee/elasticsearch-kibana
```

Now you can access Elasticsearch at http://localhost:9200 and Kibana at http://localhost:5601.

## Twitter Streaming API

In order to access the [Twitter Streaming API](https://dev.twitter.com/streaming/overview), you need to register an application at http://apps.twitter.com. Once created, you should be redirected to your app’s page, where you can get the consumer key and consumer secret and create an access token under the “Keys and Access Tokens” tab. Add these to a new file called config.py:

```Python
consumer_key = "add_your_consumer_key"
consumer_secret = "add_your_consumer_secret"
access_token = "add_your_access_token"
access_token_secret = "add_your_access_token_secret"
```

According to the [Twitter Streaming documentation](https://dev.twitter.com/streaming/overview/connecting), “establishing a connection to the streaming APIs means making a very long lived HTTP request, and parsing the response incrementally. Conceptually, you can think of it as downloading an infinitely long file over HTTP.”

So, you make a request, filter it by a specific keyword, user, and/or geographic area and then leave the connection open, collecting as many tweets as possible.

This sounds complicated, but [Tweepy](http://www.tweepy.org/) makes it easy.

## Tweepy Listener

Tweepy uses a “listener” to not only grab the streaming tweets, but filter them as well.

Save the following code as sentiment.py:
```
import json
from tweepy.streaming import StreamListener
from tweepy import OAuthHandler
from tweepy import Stream
from textblob import TextBlob
from elasticsearch import Elasticsearch

# import twitter keys and tokens
from config import *

# create instance of elasticsearch
es = Elasticsearch(["localhost:9200"], sniff_on_start=True)

class TweetStreamListener(StreamListener):

    # on success
    def on_data(self, data):

        # decode json
        dict_data = json.loads(data)

        # pass tweet into TextBlob
        tweet = TextBlob(dict_data["text"])

        # output sentiment polarity
        print(tweet.sentiment.polarity)

        # determine if sentiment is positive, negative, or neutral
        if tweet.sentiment.polarity < 0:
            sentiment = "negative"
        elif tweet.sentiment.polarity == 0:
            sentiment = "neutral"
        else:
            sentiment = "positive"

        # output sentiment
        print(sentiment)

        # add text and sentiment info to elasticsearch
        es.index(index="sentiment",
                 doc_type="test-type",
                 body={"author": dict_data["user"]["screen_name"],
                       "date": dict_data["created_at"],
                       "message": dict_data["text"],
                       "polarity": tweet.sentiment.polarity,
                       "subjectivity": tweet.sentiment.subjectivity,
                       "sentiment": sentiment})
        return True

    # on failure
    def on_error(self, status):
        print(status)

if __name__ == '__main__':

    # create instance of the tweepy tweet stream listener
    listener = TweetStreamListener()

    # set twitter keys/tokens
    auth = OAuthHandler(consumer_key, consumer_secret)
    auth.set_access_token(access_token, access_token_secret)

    # create instance of the tweepy stream
    stream = Stream(auth, listener)

    # search twitter for "congress" keyword
    stream.filter(track=['congress'])
```

What is happening?:

    We connect to the Twitter Streaming API;
    Filter the data by the keyword "congress";
    Decode the results (the tweets);
    Calculate sentiment analysis via TextBlob;
    Determine if the overall sentiment is positive, negative, or neutral;
    Finally the relevant sentiment and tweet data is added to the Elasticsearch.
    
Follow the inline comments for further details.

See [Elasticsearch Python Client API](https://elasticsearch-py.readthedocs.io/en/v7.10.1/)

### TextBlob sentiment basics

To calculate the overall sentiment, we look at the polarity score:

    Positive: From 0.01 to 1.0
    Neutral: 0
    Negative: From -0.01 to -1.0

Refer to the [official documentation](http://textblob.readthedocs.org/en/dev/) for more information on how TextBlob calculates sentiment.

## Elasticsearch Analysis

requirements.txt
```
elasticsearch==7.10.1
textblob==0.15.3
tweepy==3.10.0
```

Install dependencies
```bash
pip3 install -r requirements.txt
```

Download the twitter data into Elasticsearch
```
python3 sentiment.py
```

Over a two hour period, I pulled over 9,500 tweets with the keyword “congress”.

At this point go ahead and perform a search of your own, on a subject of interest to you. Once you have a sizable number of tweets, stop the script.

Now you can perform some quick searches/analysis.

Using the index ("sentiment") from the sentiment.py script, you can use the [Elasticsearch search API](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-search.html) to gather some basic insights.

For example:

    Full text search for “obama”: http://localhost:9200/sentiment/_search?q=obama
    Author/Twitter username search: http://localhost:9200/sentiment/_search?q=author:allvoices
    Sentiment search: http://localhost:9200/sentiment/_search?q=sentiment:positive
    Sentiment and “obama” search: http://localhost:9200/sentiment/_search?q=sentiment:positive&message=obama

There’s much, much more you can do with Elasticsearch besides just searching and filtering results. Check out the [Analyze API](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/analysis-intro.html) as well as the Elasticsearch - [The Definitive Guide](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/index.html) for more ideas on how to analyze and model your data.

## Kibana Visualizer

[Kibana](http://www.elasticsearch.org/overview/kibana/) lets “you see and interact with your data” in realtime, as you’re gathering data. Since it’s written in JavaScript, you access it directly from your browser. Check out the basics from the [official introduction](http://www.elasticsearch.org/guide/en/kibana/current/_introduction.html) to quickly get started.

We can create a pie chart in Kibana, which shows the proportion of each sentiment - positive, neutral, and negative - to the whole from the tweets I pulled.

We can build more graphs from Kibana:

    All tweets filtered by the word “obama”
	Top twitter users by tweet count
	
Aside for these charts, it’s worth visualizing sentiment by location. Try this on your own. You’ll have to alter the data you are grabbing from each tweet. You may also want to try visualizing the data with a histogram as well.
