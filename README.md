# Building a Recommender with Apache Mahout on Amazon Elastic MapReduce (EMR)
This follows the example in https://aws.amazon.com/blogs/big-data/building-a-recommender-with-apache-mahout-on-amazon-elastic-mapreduce-emr/. Since the code in the aws tutorial gives an error, the script provided here will mitigate the issue.

## Setting up the AWS cluster
1. Log into aws/ awseducate account and open the aws console
- In the console search for EMR (Elastic MapReduce) service
- Create a new cluster with Hadoop and Mahout services (recommended to create an m5.xlarge cluster to avoid issues with package and service versions.
- Wait for the cluster to start ('Running' or 'waiting' status)
- Open up PuTTY and SSH into the master node

## Setting up the test data
1.Get the MovieLens data

```
wget http://files.grouplens.org/datasets/movielens/ml-1m.zip
```
unzip the data

```
unzip ml-1m.zip
```
2.Preprocess the data for the tutorial (Convert ratings.dat, trade “::” for “,”, and take only the first three columns)

```
cat ml-1m/ratings.dat | sed 's/::/,/g' | cut -f1-3 -d, > ratings.csv
```

3.Put ratings file into HDFS:

```
hadoop fs -put ratings.csv /ratings.csv
```

## Building the recommender

```
mahout recommenditembased --input /ratings.csv --output recommendations --numRecommendations 10 --outputPathForSimilarityMatrix similarity-matrix --similarityClassname SIMILARITY_COSINE
```

## Check results
```
hadoop fs -ls recommendations
hadoop fs -cat recommendations/part-r-00000 | head
```
## Building a Service
Build a simple python service to utilize the recommender.

1.Install dependencies/ packages (twisted, klein, redis)

````
 sudo pip3_install twisted klein redis
````
> **Note**: The AWS tutorial uses easy_install which doesn't work. Which is why we use pip3 install here. You can also run it as follows

````
 sudo pip3_install twisted 
 sudo pip3_install klein 
 sudo pip3_install redis
````

2.Install Redis and start up the server.

```
        wget http://download.redis.io/releases/redis-2.8.7.tar.gz
        tar xzf redis-2.8.7.tar.gz
        cd redis-2.8.7
        make
        ./src/redis-server &
```

3. Create the python service. 
    You can use any name you like for this (ex: recom_service.py)

```
nano recom_service.py
```
This will open up a nano bash and create the file recom_service.py in your cloud service when you save your work.

Now copy the code from the https://github.com/akilapeirisz/aws-es-mahout-recommender/tree/main/script/recom_service.py
> **Note**: The issues with the script in the original aws tutorial have been fixed here.

```
from klein import run, route
import redis
import os

# Start up a Redis instance
r = redis.StrictRedis(host='localhost', port=6379, db=0)

# Pull out all the recommendations from HDFS
p = os.popen("hadoop fs -cat recommendations/part*")

# Load the recommendations into Redis
for i in p:

  # Split recommendations into key of user id 
  # and value of recommendations
  # E.g., 35^I[2067:5.0,17:5.0,1041:5.0,2068:5.0,2087:5.0,
  #       1036:5.0,900:5.0,1:5.0,081:5.0,3135:5.0]$
  k,v = i.split('\t')

  # Put key, value into Redis
  r.set(k,v)

# Establish an endpoint that takes in user id in the path
@route('/<string:id>')

def recs(request, id):
  # Get recommendations for this user
  v = r.get(id)
  return 'The recommendations for user '+str(id)+' are '+str(v)


# Make a default endpoint
@route('/')

def home(request):
  return 'Please add a user id to the URL, e.g. http://localhost:8081/1234n'

# Start up a listener on port 8081
run("localhost", 8081)
```
You can copy the content and right click on the PuTTY terminal which will paste the copied content into the file. 
Then to quit, press `Ctrl + x`. It will then ask whether to write the content on the Buffer to the file, press `Y` to do so. Then finally it will prompt the file name to amend, press `Enter`.

## Start the web service.
```
twistd -noy hello.py &
```
## Test the web service 
Access the web service via the port we provided in the python script (ex: 8081). 
Ex: To view the movie recommendations for user 5,

```
curl localhost:8081/5
```
