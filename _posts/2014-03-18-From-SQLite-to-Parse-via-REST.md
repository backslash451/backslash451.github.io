---
layout: post
title: "From SQLite to Parse via REST"
description: "A quick tutorial on importing existing data to Parse backend via REST Api"
category: articles
tags: [backend, database, mobile, parse]
image:
  feature: cloud.jpg
comments: true
share: true
---

Recently, while working on a iOS app, I came across the problem of **how to import existing data into the [Parse](http://www.parse.com/) backend from an existing database, keeping relationships between tables**. Here you cand find the story of how i have dealt with the problem and what kind of solution i came up with.

For those of you who don’t know Parse let me say that it is a super quick backend solution for mobile (not only) applications that lets you be up and running in minutes. If you want to know more about Parse, please read their awesome [tutorials](https://parse.com/docs/index), or this great [guide](http://www.raywenderlich.com/19341/how-to-easily-create-a-web-backend-for-your-apps-with-parse) by Antonio Martinez.

<figure>
	<img src="{{ site.url }}/images/schermata-2013-11-07-alle-16-39-57.png">
</figure>

So i had this very useful website full of data but still with this old fashioned ’90 look and i decided to grab its data via a Perl script to fill up a SQLite database. What was i supposed to do with that data? That’s another story (or blog post maybe) but [here](https://gist.github.com/backslash451/7357077) you have an excerpt of that Perl script. The script use [LWP::Simple](http://search.cpan.org/~gaas/libwww-perl-6.05/lib/LWP/Simple.pm) to get the webpage, [HTML::TreeBuilder](http://search.cpan.org/~cjm/HTML-Tree-5.03/lib/HTML/TreeBuilder.pm) to parse the html and [DBI](http://search.cpan.org/~timb/DBI/DBI.pm) to connect to the SQLite3 database.

## First of all Google for it…

And here is where it all starts… so these are the steps i made to import that data to the Parse backend:

1. Check the Parse [tutorials](https://parse.com/docs/index). Here i found that you can import existing data into Parse using CSV or JSON files or by their REST api. So i tried first with CSV with no results, then with JSON using this tool. Idem. The problem is with relationship. There is no way to upload data into Parse keeping exising relationship using files. You should use REST api.
2. Check the Parse [documentation](https://parse.com/help). No answer other than pt.1 or “try [mysql2parse](https://github.com/gfosco/mysql2parse)” (more [here](https://parse.com/questions/import-mysql-via-csv-and-keeping-relationships)). So i converted my SQLite db to MySQL and tried mysql2parse. But it didn’t work. Damn! The script is very well written but it uses Basic auth while you should now use the Parse REST key to authenticate.
3. Modified mysql2parse to use the REST key instead of the Basic auth. Nothing. It works for a while but then it crashes.

## …Then write your own code

All of these steps took me approximately half a day. **So i decided to write my own code to solve my own problem. And that was the best choice ad you will see**. [Here](https://gist.github.com/backslash451/7333357) is the complete code i wrote. Please note that this is my very first script in Python and probably it is highly inefficient, ugly etc… but it gets the job done. I had just 3 tables in the DB and the script on Gists is just an excerpt but you should figure it out.

The script first connect to the DB and get a cursor to create queries:

{% highlight python %}
{% raw %}
#!/usr/bin/env python
 
import sqlite3, json, httplib, urllib
 
#  Main ----------------
conn = sqlite3.connect('products.sqlite')
c = conn.cursor()
{% endraw %}
{% endhighlight %}

Then for every row in the table “producer” create an object on Parse via REST adding an “oldID” attribute equal to the table id. We will use the oldID attribute to find out the producer which is in relation with our product:

{% highlight python %}
{% raw %}
# insertProducer -----------------------------------------------------
def insertProducer(f):
    connection = httplib.HTTPSConnection('api.parse.com', 443)
    connection.connect()
 
    print("+ {0}", f[1].encode('utf-8').strip())
 
    data = {
        'id': f[0],
        'oldID': f[0],         
		'name': f[1].encode('utf-8').strip(),
        'type': f[2],
    }
 
    if f[3]:
        data['logo'] = f[3].encode('utf-8').strip()
 
    if f[4]:
        data['companyName'] = f[4].encode('utf-8').strip()
 
    connection.request('POST', '/1/classes/producer', json.dumps(data),
         {
           "X-Parse-Application-Id": "YOUR PARSE APP ID",
           "X-Parse-REST-API-Key": "YOUR PARSE REST API KEY",
           "Content-Type": "application/json"
         })
    result = json.loads(connection.getresponse().read())
    print result
 
# Create a Parse Object for every row found in the table 'producer'
for f in c.execute('SELECT * FROM producer ORDER BY id'):
    insertProducer(f)
{% endraw %}
{% endhighlight %}

Now is time to insert our products into Parse with their relationship with producer objects! In order to do this, we first query the products table, then for each product we look for the foreign key which links it to its producer and we query Parse to get the producer object that has the same “oldID” attribute of our foreign key! Let’s do this:

{% highlight python %}
{% raw %}
# retrieveProducer -------------------------------------------------------------------
def retrieveProducer(idF):
    connection = httplib.HTTPSConnection('api.parse.com', 443)
    params = urllib.urlencode({"where":json.dumps({
           "oldID": idF
         })})
    connection.connect()
    connection.request('GET', '/1/classes/producer?%s' % params, '', {
           "X-Parse-Application-Id": "YOUR PARSE APP ID",
           "X-Parse-REST-API-Key": "YOUR PARSE REST API KEY",
         })
    result = json.loads(connection.getresponse().read())
    # print result
    return result['results'][0]['objectId']
    # return result['results'][0]
 
# insertProduct ----------------------------------------------------------------------
def insertProduct(s, idProducer):
    connection = httplib.HTTPSConnection('api.parse.com', 443)
    connection.connect()
 
    connection.request('POST', '/1/classes/product', json.dumps({
       "id": s[0],
       "oldID": s[0],
       "color": s[1].encode('utf-8').strip(),
       "producer": {"__type":"Pointer","className":"producer","objectId":idProducer}
     }),
         {
           "X-Parse-Application-Id": "YOUR PARSE APP ID",
           "X-Parse-REST-API-Key": "YOUR PARSE REST API KEY",
           "Content-Type": "application/json"
         })
    result = json.loads(connection.getresponse().read())
    print result
 
# For each product found in the table 'produuct':
#   retrieve the correspondant producer PObject
#   create a Parse Object with the relationship with producer
for s in c.execute('SELECT * FROM product ORDER BY id'):
   idProduct = s[2]
   producer = retrieveProducer(idProduct)
   insertProduct(s, producer)
{% endraw %}
{% endhighlight %}

And that’s it!

## Conclusions

I solved my problem with this script but, as you can see, it is neither efficient nor adaptable to wider contexts where you have a greater number of tables.

One of the main drawbacks of this approach is the huge number of api call to Parse backend, and this could be a problem when the free account has a 1 Million api call limit. You should probably do some maths before using this.  :)

<figure>
	<img src="{{ site.url }}/images/schermata-2013-11-08-alle-12-00-20.png">
</figure>
