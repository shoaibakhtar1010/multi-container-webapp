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

