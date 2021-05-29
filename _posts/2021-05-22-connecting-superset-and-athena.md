---
title:  "Connecting Superset and Amazon Athena"
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
If you like me are running Windows 10 you need to install Superset in Ubuntu on Windows. Using [pip](https://pypi.org/project/pip/) install Superset like so:

{% highlight console %}
pip install apache-superset
{% endhighlight %}

In order to make Superset "talk" to Athena install [PyAthena](https://pypi.org/project/pyathena/) which is a Python client for Athena:

{% highlight console %}
pip install PyAthena
{% endhighlight %}

Initialize the database:

{% highlight console %}
superset db upgrade
{% endhighlight %}

Now you're ready to fire up Superset:

{% highlight console %}
superset run -p 8088
{% endhighlight %}

and go to http://localhost:8088

In the Superset web UI select Databases:

![title](/assets/images/superset-databases.JPG)


