---
title:  "Connecting Superset and Amazon Athena"
permalink: connecting-superset-and-athena
description: How to connect Superset and Amazon Athena
date:   2021-05-17 13:19:13 +0200
categories: 
  - blog
tags:
  - amazon athena
  - superset
---
[Apache Superset](https://superset.apache.org/) is an open-source business intelligence tool that can be connected with many different data sources. 
[Amazon Athena](https://aws.amazon.com/athena) is a query engine built on top of [Presto](https://prestodb.io/) and can be used to analyze data stored in S3.

This post shows how to use Amazon Athena as a data source.

## Install Superset
If you're running Windows 10 I recommend installing Superset in Ubuntu on Windows. 


Start installing prerequisites:

{% highlight console %}
sudo apt update
sudo apt install build-essential libssl-dev libffi-dev python3-dev python3-pip libsasl2-dev libldap2-dev python3-venv
{% endhighlight %}

Create a virtual environment for the Superset installation:

{% highlight console %}
python3 -m venv superset
source superset/bin/activate
{% endhighlight %}

The following may not be needed but I had to change version of the following dependencies like so:

{% highlight console %}
pip install "SQLAlchemy<1.4.0"
pip install "itsdangerous<2.0,>=0.24"
{% endhighlight %}

Now install Superset:

{% highlight console %}
pip install apache-superset
{% endhighlight %}

In order to make Superset "talk" to Athena install [PyAthena](https://pypi.org/project/pyathena/) which is a Python client for Athena:

{% highlight console %}
pip install PyAthena[SQLAlchemy]
{% endhighlight %}

Now initialize the database:

{% highlight console %}
superset db upgrade
{% endhighlight %}

and create an admin user:

{% highlight console %}
export FLASK_APP=superset
superset fab create-admin
{% endhighlight %}

and finally, setup roles and permissions:
{% highlight console %}
superset init
{% endhighlight %}

Now you're ready to fire up Superset:

{% highlight console %}
superset run -p 8088
{% endhighlight %}

and go to http://localhost:8088

In the Superset web UI select Databases:

![title](/assets/images/superset-databases.JPG)

and create new database:

![title](/assets/images/createdatabase.JPG)

In the add database dialog, name your database and add a connection string with appropriate values:

{% highlight console %}
awsathena+rest://<aws-access-key-id>:<aws-secret-access-key>@athena.eu-west-1.amazonaws.com/etldatabase__prod?s3_staging_dir=<uri-of-staging-directory>
{% endhighlight %}

Now open SQL Lab -> SQL Editor on you're ready to query your Glue database in Superset!