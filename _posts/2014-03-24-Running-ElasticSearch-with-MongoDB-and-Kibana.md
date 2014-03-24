---
layout: post
title: "Running Elasticsearch with MongoDB"
description: "A short tutorial on running Elasticsearch with MongoDB and Kibana"
category: articles
tags: [backend, database, search, mongodb, elasticsearch, kibana, river]
image:
  feature: elasticsearchHeader.png
comments: true
share: true
---

## Introduction
This is just a short tutorial to get up and running in a matter of minutes with [**Elastisearch**](http://www.elasticsearch.org/overview), [**MongoDB**](http://www.mongodb.com/) and [**Kibana**](http://www.elasticsearch.org/overview/kibana/). What i want to achieve is having a MongoDB instance with some data in a certain collection, some automatic mechanism to synch these data with Elasticsearch (which as you will see is something called "**[River](http://www.elasticsearch.org/guide/en/elasticsearch/rivers/current/ "Rivers")**") and a simple and configurable way to present and query these data.

## Preparing MongoDB
Installing MongoDB is out of the scope of this tutorial, but if you need support on this you can find detailed instructions [here](http://docs.mongodb.org/manual/installation/ "Install MongoDB &mdash; MongoDB Manual 2.4.9").

What we want to do is configure MongoDB with a [**ReplicaSet**](http://docs.mongodb.org/manual/replication/ "Replication &mdash; MongoDB Manual 2.4.9") even if we are using a standalone instance. The reason for this is that the Elasticsearch plugin depends on the [**oplog**](http://docs.mongodb.org/manual/core/replica-set-oplog/ "Replica Set Oplog &mdash; MongoDB Manual 2.4.9") (a log of all changes used by MongoDB to replicate itself) to push new updates into Elastisearch.

In order to configure MongoDB with a ReplicaSet you can follow their [tutorial](http://docs.mongodb.org/manual/tutorial/convert-standalone-to-replica-set/ "Convert a Standalone to a Replica Set &mdash; MongoDB Manual 2.4.9") or you can follow this simple procedure:

### Create data directories
First thing to do is to create 3 empty directories that will be used by each MongoDB node to store its data:

{% highlight bash %}
$ mkdir <PATH>/node1
$ mkdir <PATH>/node2
$ mkdir <PATH>/node3
{% endhighlight %}

### Start Mongod instances
Simply start each mongodb node giving it a replica set name, individual port and the data directory you just created:

{% highlight bash %}
$ mongod --replSet test --port 27017 --dbpath <PATH>/node1
$ mongod --replSet test --port 27018 --dbpath <PATH>/node2
$ mongod --replSet test --port 27019 --dbpath <PATH>/node3
{% endhighlight %}

### Configure the Replica Set
Now that the mongodb nodes are up and running, the next step is to configure the Replica Set through the mongo command shell:

{% highlight bash %}
$ mongo
MongoDB shell version: 2.4.4
connecting to: test
Server has startup warnings: 
Mon Mar 24 10:14:10.114 [initandlisten] 
Mon Mar 24 10:14:10.114 [initandlisten] ** WARNING: soft rlimits too low. Number of files is 256, should be at least 1000
> config = {_id: 'test', members: [ {_id: 0, host: 'localhost:27017'}, {_id: 1, host: 'localhost:27018'}, {_id: 2, host: 'localhost:27019'}]};
> rs.initiate(config);
{% endhighlight %}

You can verify that everything is up and runnig correctly with this command:

{% highlight bash %}
> rs.status();
{% endhighlight %}

### Import some data
This step is optional but it is useful to import some data significant to play with in MongoDB. I used the **Enron Email Corpus** dataset wich contains 1,5 GB of MongoDB data, comprising 517425 emails. [Here](http://mongodb-enron-email.s3-website-us-east-1.amazonaws.com/ "Enron Email Corpus for MongoDB") you can find detailed instructions on how to download it and import it in MongoDB using the **mongorestore** tool.

{% highlight bash %}
$ mongorestore --db enron --collection messages messages.bson
{% endhighlight %}

{% highlight json %}
{
	"_id" : ObjectId("4f2ad4c4d1e2d3f15a000000"),
	"body" : "Here is our forecast\n\n ",
	"subFolder" : "allen-p/_sent_mail",
	"mailbox" : "maildir",
	"filename" : "1.",
	"headers" : {
		"X-cc" : "",
		"From" : "phillip.allen@enron.com",
		"Subject" : "",
		"X-Folder" : "\\Phillip_Allen_Jan2002_1\\Allen, Phillip K.\\'Sent Mail",
		"Content-Transfer-Encoding" : "7bit",
		"X-bcc" : "",
		"To" : "tim.belden@enron.com",
		"X-Origin" : "Allen-P",
		"X-FileName" : "pallen (Non-Privileged).pst",
		"X-From" : "Phillip K Allen",
		"Date" : "Mon, 14 May 2001 16:39:00 -0700 (PDT)",
		"X-To" : "Tim Belden ",
		"Message-ID" : "<18782981.1075855378110.JavaMail.evans@thyme>",
		"Content-Type" : "text/plain; charset=us-ascii",
		"Mime-Version" : "1.0"
	}
}
{% endhighlight %}

## Installing Elasticsearch
Now that you have your MongoDB nodes running and a bunch of data stored in the enron database you are ready to install Elasticsearch. As stated by [their page](http://www.elasticsearch.org/overview/elkdownloads/ "Elasticsearch.org Download ELK | Elasticsearch") nothing simpler:

1. Download and unzip the latest Elasticsearch distribution
2. Run *bin/elasticsearch* to start the es server.
3. Run *curl -XGET http://localhost:9200/* to confirm it is working.

...and that's it!

## Installing Kibana
Kibana is just a bunch of javascript and html files. So "installing" may not be the best word to use but here it is your shop list: 

1. Download and unzip the latest Kibana distribution (3.0.0 at the time of writing)
2. Edit the *config.js* file setting the *elasticsearch* parameter to the fully qualified hostname of your Elasticsearch server (in this case elasticsearch: "http://localhost:9200" )
3. Copy the contents of the extracted directory to your webserver
	* If you don't have a webserver you can simply use this usefull python package to run a webserver with a single line command: 
	 
		*$ python -m SimpleHTTPServer 1234*
4. Point your browser at your shiny new installation (http://localhost:1234)

... and (again) that's it!

## Installing and configuring the MongoDB River
Latest thing to do is configuring the [**River for MongoDB**](https://github.com/richardwilly98/elasticsearch-river-mongodb) with the [**Mapper Attachments Type for Elasticsearch**](https://github.com/elasticsearch/elasticsearch-mapper-attachments).

{% highlight bash %}
$ bin/plugin --install com.github.richardwilly98.elasticsearch/elasticsearch-river-mongodb/2.0.0
$ bin/plugin --install elasticsearch/elasticsearch-mapper-attachments/2.0.0.RC1
{% endhighlight %}

Once this is done, you need to actually create the "River" and the Index:

{% highlight bash %}
$ curl -XPUT 'http://localhost:9200/_river/mongodb/_meta' -d '{ 
	"type": "mongodb", 
	"mongodb": { 
		"db": "enron", 
		"collection": "messages"
	},
	"index": {
		"name": "enron", 
		"type": "messages" 
	}
}' 
{% endhighlight %}

Now you can run queries to Elastisearch through *curl* commands to verify it is working (install the [Node](http://nodejs.org/ "node.js") [jsontool](http://nodetoolbox.com/packages/jsontool "The Node Toolbox - jsontool") if you want to visualize json in a pretty indented way in you bash):

{% highlight bash %}
$ npm install -g jsontool
$ $curl -CGET 'http://localhost:9200/enron/_search?q=headers.To:"ebass@enron.com"' | json -i
{% endhighlight %}

## Conclusions
Now, assuming:

1. You followed every step since now
2. MongoDB ReplicaSet is up and running
3. Elasticsearch if up and running
4. MongoDB River is configured
5. Kibana is configured and deployed in a running web server

This is what you should see if you point your browser to [http://localhost:1234/](http://localhost:1234/) and configure some panels on Kibana:

<figure>
	<img src="{{ site.url }}/images/Schermata 2014-03-24 alle 11.51.11.png">
</figure>