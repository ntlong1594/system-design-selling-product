# system-design-selling-product

# Given Scenario
We are going to develop a commercial website for selling products where:
- Each product has a number of available items in stock.
- The system should be able to process at least N = 6000 concurrent requests for viewing
or purchasing products.
- The system is only allowed to have at most S = 6 servers, where up to 3 servers can be
used as relational databases.
- One relational database can serve at most C = 300 concurrent connections.
- The system must: (main requirements)
- ensure data consistency, which means that there will be never such a case
where the number of successful purchased items is more than the number of
available items in stock.
- provide real-time feedback to its users with the lowest possible latency.
- It will be good to: (bonus requirements)
- have the highest availability with no point of failure.
- be easily scalable in the future when the number of products and users
increase.

# Overview
This system is a kind of ecommerce project, in this excersice, we will provide an high level architecture
and component diagram  and microservices design is well fit with this scenario and java will be our main language.

# Technical using
- Angular2+, Spring Boot framework
- Reactive Java - RxJava (support non-blocking I/O, asynchronous request)
- Message Queue (Apache Kafka)
- Cache Server (Redis)
- NoSql (Mongodb)
- Searching engine (ElasticSearch)
- Discovery Client (service monitor)
- Gateway
- Authenticate server (Keycloak)
- Docker (container deployment)
- Config server (Spring cloud config)

# High-level architecture
![](imgs/system-design.svg)
- Webview will be the entry point that customer access to our system.
- The gateway will be the middleman to help to communicate between Frontend and Backend as well as communication between microservice and microservice.
gateway will let us centralized the request before dispatching it to destination service. Besides that, 
By applying gateway, when the number of service's instances scale up, we don't need to take to much effort to re-config it.
- Let say we use Keycloak as an authenticate server, this server will support multiple type of authenticate such as: oauth, basic login, ... they will provide
login token when authenticate success for respective service.
- Service Discovery Client will help us to monitor microservice and its own running instances, and cloud config server will be take a role of centralize all config of 
the eco system. The service instead of loading its own configuration locally, it will connect to Config Server and retrieve corresponding info.
- Product Service is our main service in this design , it will support browsing product and make purchasing which are the two key functions we need to handle.

# Solving Detail
User could be browsing and make purchasing in our system. 
As given Scenario, we are dealing with 6,000 concurrent request but our database can handle maximum 900 request (after scaling 3 instances). 
Therefore,keep in mind that we need to reduce the number of request to DB as much as possible but must maintain the consistency of data record in database and 
reduce the latency of the response to user.
We will go from top to the bottom and enhance the system to make it more robust and can handle large request. 
`Please always keep in mind that all of request will apply RxJava (non-blocking I/O) instead of traditional rest template flow.`

We are provided 6 servers, let divided into:
- 1 server for elasticsearch service
- 2 server for mongodb
- 2 server for product services
- 1 server for gateway

### Webview
- In this layer, we just simply avoid user click on search button and make purchase many times in short period. 
Let's show waiting icon or disable the button for few seconds after user click.

### Gateway
- In gateway, Our strategy will apply `memory caching` for some specify request which mean if the same user send the same request within short-time, 
instead of connecting to our service we will retrieve it from our memory cache. 
In this design, I use Hystrix which support us caching request just by few line of config. By default, Hystrix will cache for 30s

### Product service
- Product service is the main service therefore, it will handle most of the request in this layer.
- Let's break the request into 2 types and trying to resolve one by one: 
    - Browsing product ( searching , view detail, ...)  
    - Make purchase  
 
#### Browsing product
- To the product selling system, The requests to search product are frequently. To reduce workload for mongodb, 
instead querying directly to mongodb, `Elasticsearch` would be apply. Elasticsearch is fast for querying. 
To be use `Elasticsearch` parallel with Mongodb and keep the data always consistency and up to date, we need to have a mechanism to synchronized both of them. 
But in this design I won't go to detail about it
- Whenever, new product added or product updated it will update to both MongoDB and Elasticsearch. 
- But for searching product, Elasticsearch could take a better performance than MongoDB. 

#### Make purchase
- This request is the most complicated we need to handle because we need to handle large amount of request as well as
 ensure the consistency of data which mean don't let user buy the quantity of product more than what we have in our stock.
 
- Let talk about the Database collection structure first. To ensure data consistency , we will apply `Optimistic Locking` in our collection,
which mean each record will mark a version, before updating, system will check against the version to ensure data is the newest otherwise we will throw an exception.

- To be ensure the product quantity up to date, in this situation, we will load the product with its own quantity to `cache server` , 
in this case `Redis` is one of a good option. Redis is key-value stored, therefore, the key is stand for SKU-ID (product ID) 
and the value is atomic number which ensure data integrity when concurrent request decrement the value at the same time. 
Besides that, Redis could be apply to store user's shopping cart temporary

We just apply `Optimistic locking` and `Redis caching` , these two options just help us ensure the data integrity and data consistency between when a lot of requests comming.

Now let discuss about the process of making purchase. I assume that making purchase is complex and take very long time to process. 
Therefore, when user click on purchase , instead of letting them waiting for the response immediately. 
`Let apply Message Queue ( Kafka) to process and let user tracking their purchase status`. 

Applying Kafka,not only support us to process data in order but also support us on failover with DLQ handling.
Take a look on below flow:

![](imgs/process-event.svg)

Let say we provide 2 simple statuses for the purchases: RECEIVED, CONFIRMED / FAILED. While data streaming process, 
in case of failure, the message will be publish to `retry-purchase-channel`, 
there is a consumer will be consume it and process our retry strategy. After x times retry , the message will be published to
`failed-purchase-channel` and notify to our customer or stakeholder

# Conclusion
- With the above design, the requests are already eliminated from the very top layer (webview, gateway). 
Our product services just need to focus processing important and new request. 
By applying ElasticSearch and Message Queue, the nubmer of requests from product service to Database are reducing, 
therefore, our database could adapt the remaining process well.
