# docker-multi-container-webapp
This is a project that uses docker and aws EBS to build a multi container application.

![Image](https://im.ezgif.com/tmp/ezgif-1-d79e09c7bb.gif)

The above diagram is a development architecture of our application. When the user tries to use our app through browser they're gonna first visit an Nginx server.They nginx server will do some routing and its going to decide whether the browser is trying to access the react app to get some frontend assets like the html file or some js file that will be used to build this app. If the incoming request is trying to access some backend api that we are going to use for submitting numbers, reading numbers and retrieving values then nginx server will route the request to an express server. The express server is going to function as an api that's going to serve up info or calculated values up to the frontend app.

![Image](https://1.bp.blogspot.com/sgCAajBO3FBWpmcoWGQQ2yfvLXfiMrAmiIOl-LiRETGlfo08_uIKl6C_ElV0n1KP674rAFXUPb3VUps=w620-h391)

# Redis
Redis is in memory data store and its very commonly used for housing temp or kind of cached values. Calculated values that are going to be displayed, all this info is going to be stored in a separate redis db.

# Postgres
 used as the primary data store or data warehouse for many web applications. Values I have seen is very permenant stored set of data so its stored in postgres db.
 
 
![Image](https://1.bp.blogspot.com/RY3xZBdfnDKx2uqM9w8P7l34NueVIbbul3tQyILRUB1BfHCRb1ToiR1WSxAuz776D725NnxtulX4ovU=w602-h307)

So when the user clicks on that submits button, the react app is going to make an API request to backend expresss server. The express server when receive this number, that it needs to calculate fib number for its gonna first take that number and store it inside our Pg DB which is having a permenant list of all the indices that have ever been submitted to our app at the same time the exprewss server is going to take that index and put it into the redis DB as well. When a new number shows up inside the redis DB, its gonna wakeup a second backend node.js process that we are going to refer as the worker. The only job of this worker here is to watch redis for new indices that show up, Anytime a new index shows up inside of redis the worker is going to pull rthat value out, it will calculate the appropriate fibonacci value for it and it will take that calculated value and then it put it back into redis. so then it can then be requested by the react application.

# purpose of nginx
![Image](https://1.bp.blogspot.com/dfONAOUvKvLqziPuK11lQAp71QGRm6CZJgz_CuDc5nFiYALgl8W1asiZldqzFoVpUkDkCK3-zHaM0PA=w654-h325)

In the above dig we got a browser on LHS and 2 different servers that exist inside of our dev environment. Next to the browser is a list of some of the different requests that the browser is going to make to our environment. So the browser at some point of time is gonna ask for an index.html file. It needs an html file to show on the screen of browser and when it loads up that html file its probably gonna need a js file as well like some file that has all the js code for the react side of our app. In addition to that we know the browser is going to make some api requests to the express server to get the list of all of the different values that have ever been submitted and the list of all the different current values that have been calculated by the worker process.

  So the index.html and js requests need to go over to the react server, but the api requests need to go over to the express server. so the purpose of nginx sever is gonna look at all the requests coming from the browser and decides on which backend service of ours we want to route the request to. In the server directory we have index.js file inside of here all the route handlers are like /values/all or /values/current or /values. However in the client src.js file we are making network requests to routes of /api/values/current, /api/values/all. so the client think that it needs to /api but its very clear that the server is not setup to receive that /api route.
  
  ![Image](https://1.bp.blogspot.com/Wb5O3FkDwQFZzBMNadxFs4ZpT2cO-cRhUVVhjXL0AwpD0NC3udbloXq2sz2OK73xgTQgRXSVTa_57xI=w641-h273)
  
   We are gonna add an additional container to our app by adding in a service to the docker-compose file. That additional service is gonna startup the nginx server and the nginx server is going to have essentially one job, i.e any time a request comes to nginx server its gonna look at the incoming request more specifically request path. If /api in the route that this request is trying to access , then the nginx redirect this request over to the express serverotherwise if it is not looking for something li8ke /api the nginx redirect that request over to react server instead. If you look at server directory and find index./js inside therew you will see all of our requests handlers here are not specifying a /api on the very front of it. The reason fot that is that after nginx does this little routing step its then going to chop off the /api part of the route. so something like /api/values/all comes to nginx and when it comes out of it its simply /values/all.

 ![Image](https://1.bp.blogspot.com/5kSqMFhN9KUaNvwyjTyq6ityVxLevQSzcX9nFvGz1ejywsel2us3TO4hdZa8jXSzYALiQ2sv9z80jjk=w620-h347)

we are going to build a default.conf config file for nginx routing. Inside this config file we are going to setup all the instruction that we see on RHS of the image above. First off, we are gonna tell nginx that there is an upstream server at client:3000 and server:5000. Nginx is going to watch for requests from the outside world and then route them to appropriate servers. These servers are kind of behind nginx, you can not access these servers unless you go through nginx server. so nginx refers to these as upstream servers, they are servers that nginx can optionally redirect traffic over to. In both statement we can notice a port has been assigned where we can find upstream server i.e there is a server at client:3000 and server:5000. So when we put together our initial server implementation, inside the server directory we find the index.js file at the very bottom of it we find a server listening on port 5000. so the server is watching for traffic on port 5000 and a client has a create react app by default listens on port 3000. so the client:5000 and server:3000 are actual addresses. so client and server are actual host names that nginx is try to redirect traffic to.

After we put together those 2 quick definitions, we'll then tell nginx that it's going to listen by default on port 80. After that we setup the 2 rules that say essentially if anyone comes to '/' or root route we wanna send them to the client upstream  and if anyone goes to /api send them to the server upstream.


![Image](https://1.bp.blogspot.com/EXgwyLQgnUZha8Rof_n4PUHm4LOhePNE27VL5WIEOKdYkk0Fbjv6XYLG_oJDD4Q6M6hj4-RovFuMVxo=w648-h288)

With multi container setup specifically in dev env, the browser will make request into the initial nginx server and this nginx server is specifically responsible for routing and making sure request get to the correct backend. Now in production, we still have nginx server solely dedicated vfor serving up prod react files but this time inside EBS instance its going to be listen on port 3000 and users are only able to access this nginx server with all prod files by first going through the other copy of nginx that is specifically responsible for routing. so we have done a little config of nginx server to make sure that it instead listens on port 3000 and still exposes all the react production assets on port 3000 instead. 





# AWS Configuration Cheat Sheet - Updated for new UI
updated 9-28-2022

This lecture note is not intended to be a replacement for the videos, but to serve as a cheat sheet for students who want to quickly run thru the AWS configuration steps or easily see if they missed a step. It will also help navigate through the changes to the AWS UI since the course was recorded.

## EBS Application Creation (If using Multi-Container Docker Platform)

Go to AWS Management Console and use Find Services to search for Elastic Beanstalk

Click “Create Application”

Set Application Name to 'multi-docker'

Scroll down to Platform and select Docker

In Platform Branch, select Multi-Container Docker running on 64bit Amazon Linux

Click Create Application

You may need to refresh, but eventually, you should see a green checkmark underneath Health.

## EBS Application Creation (If using Amazon Linux 2 Platform Platform)

Make sure you have followed the guidance in this note.

Go to AWS Management Console and use Find Services to search for Elastic Beanstalk

Click “Create Application”

Set Application Name to 'multi-docker'

Scroll down to Platform and select Docker

The Platform Branch should be automatically set to Docker Running on 64bit Amazon Linux 2.

Click Create Application

You may need to refresh, but eventually, you should see a green checkmark underneath Health.

RDS Database Creation

Go to AWS Management Console and use Find Services to search for RDS

Click Create database button

Select PostgreSQL

Change Version to the newest available v12 version (The free tier is currently not available for Postgres v13)

In Templates, check the Free tier box.

Scroll down to Settings.

Set DB Instance identifier to multi-docker-postgres

Set Master Username to postgres

Set Master Password to postgrespassword and confirm.

Scroll down to Connectivity. Make sure VPC is set to Default VPC

Scroll down to Additional Configuration and click to unhide.

Set Initial database name to fibvalues

Scroll down and click Create Database button

## ElastiCache Redis Creation

Go to AWS Management Console and use Find Services to search for ElastiCache

In the sidebar under Resources, click Redis Clusters

Click the Create Redis cluster button

Make sure Cluster Mode is DISABLED.

Scroll down to Cluster info and set Name to multi-docker-redis

Scroll down to Cluster settings and change Node type to cache.t2.micro

Change Number of Replicas to 0 (Ignore the warning about Multi-AZ)

Scroll down to Subnet group settings and select an available subnet group.

Scroll down and click the Next button

Scroll down and click the Next button again.

Scroll down and click the Create button.

## Creating a Custom Security Group

Go to AWS Management Console and use Find Services to search for VPC

Find the Security section in the left sidebar and click Security Groups

Click Create Security Group button

Set Security group name to multi-docker

Set Description to multi-docker

Make sure VPC is set to your default VPC

Scroll down and click the Create Security Group button.

After the security group has been created, find the Edit inbound rules button.

Click Add Rule

Set Port Range to 5432-6379

Click in the box next to Source and start typing 'sg' into the box. Select the Security Group you just created.

Click the Save rules button

## Applying Security Groups to ElastiCache

Go to AWS Management Console and use Find Services to search for ElastiCache

Under Resources, click Redis clusters in Sidebar

Check the box next to your Redis cluster

Click Actions and click Modify

Scroll down to find Selected security groups and click Manage

Tick the box next to the new multi-docker group and click Choose

Scroll down and click Preview Changes

Click the Modify button.

## Applying Security Groups to RDS

Go to AWS Management Console and use Find Services to search for RDS

Click Databases in Sidebar and check the box next to your instance

Click Modify button

Scroll down to Connectivity and add select the new multi-docker security group

Scroll down and click the Continue button

Click Modify DB instance button

## Applying Security Groups to Elastic Beanstalk

Go to AWS Management Console and use Find Services to search for Elastic Beanstalk

Click Environments in the left sidebar.

Click MultiDocker-env

Click Configuration

In the Instances row, click the Edit button.

Scroll down to EC2 Security Groups and tick the box next to multi-docker

Click Apply and Click Confirm

After all the instances restart and go from No Data to Severe, you should see a green checkmark under Health.

Add AWS configuration details to .github/workflows/deploy.yaml file's deploy script

Set the region. The region code can be found by clicking the region in the toolbar next to your username.
eg: 'us-east-1'

app should be set to the EBS Application Name
eg: 'multi-docker'

env should be set to your EBS Environment name.
eg: 'MultiDocker-env'

Set the bucket_name. This can be found by searching for the S3 Storage service. Click the link for the elasticbeanstalk bucket that matches your region code and copy the name.

eg: 'elasticbeanstalk-us-east-1-923445599289'

Set the bucket_path to 'docker-multi'

Set access_key_id to $AWS_ACCESS_KEY

Set secret_access_key to $AWS_SECRET_KEY

## Setting Environment Variables

Go to AWS Management Console and use Find Services to search for secret manager

Click store a new secret button in the right.

select other type of secret  --> key/value


In another tab Open up ElastiCache, click Redis and check the box next to your cluster. Find the Primary Endpoint and copy that value but omit the :6379

Set REDIS_HOST key to the primary endpoint listed above, remember to omit :6379

Set REDIS_PORT to 6379

Set PGUSER to postgres

Set PGPASSWORD to postgrespassword

In another tab, open up the RDS dashboard, click databases in the sidebar, click your instance and scroll to Connectivity and Security. Copy the endpoint.

Set the PGHOST key to the endpoint value listed above.

Set PGDATABASE to fibvalues

Set PGPORT to 5432

Click Apply button

## After all instances restart and go from No Data, to Severe, you should see a green checkmark under Health.

## IAM Keys for Deployment

You can use the same IAM User's access and secret keys from the single container app we created earlier, or, you can create a new IAM user for this application:

1. Search for the "IAM Security, Identity & Compliance Service"

2. Click "Create Individual IAM Users" and click "Manage Users"

3. Click "Add User"

4. Enter any name you’d like in the "User Name" field.

eg: docker-multi-container

5. Tick the "Programmatic Access" checkbox

6. Click "Next:Permissions"

7. Click "Attach Existing Policies Directly"

8. Search for "beanstalk"

9. Tick the box next to "AdministratorAccess-AWSElasticBeanstalk"

similarly Search for "secrets manager"

Tick the box next to "Secrets Manager Read/write"

10. Click "Next:Tags"

11. Click "Next:Review"

12. Click "Create user"

13. Copy and / or download the Access Key ID and Secret Access Key to use in the Travis Variable Setup.

AWS Keys in Actions

Go to your github  --> Your repo  -->  settings  --> secrets and add  


Create an AWS_ACCESS_KEY variable and paste your IAM access key

Create an AWS_SECRET_KEY variable and paste your IAM secret key

Deploying App

Make a small change to your src/App.js file in the greeting text.

In the project root, in your terminal run:

git add.
git commit -m “testing deployment"
git push origin main
Go to your actions dashboard in github repo Dashboard and check the status of your build.

The status should eventually return with a green checkmark and show "build passing"

Go to your AWS Elasticbeanstalk application

It should say "Elastic Beanstalk is updating your environment"

It should eventually show a green checkmark under "Health". You will now be able to access your application at the external URL provided under the environment name.
