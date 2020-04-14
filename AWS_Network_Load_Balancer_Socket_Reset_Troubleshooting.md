## Overview
This page discusses a java socket connection reset exception discovered during migration from AWS ELB(Elastic Load Balancing) to NLB(Network Load Balancer) in preview environment.
In this environment, we have monolithic application and many microservices which run with managed Kubernetes(k8s) cluster. 
The AWS ELB sits between monolithic and k8s cluster. Recently, we migrated this infrastructure to replace ELB as NLB. 
Since NLB offers higher performance, and we also want to evaluate this offering.
## Problem
After completing this migration, our QA engineers saw unusual HTTP 503 error responses from monolithic log, and many "java.net.SocketException: Connection reset" exceptions were thrown when monolithic sent HTTP requests to microservicres. 
This kinds of exceptions occurred in every microservice. The socket exception was thrown right away after sending the request to micro service.
```
INFO [2020-04-10T16:33:55,873] [appserver-a1.na1dc1us.xxxxx.com:hystrix-xxxxxSearchService-8:2A08ED314C.200410163355836.895] 
[xreqid=3edbe2a0-bcdd-45fd-8251-af988f8d09ba] (XXXXHttpProxy.handle:85) Making proxy request to https://xxxxSearchService-na1us.micro.xxxx.com with headers {...}

ERROR [2020-04-10T16:33:55,877] [appserver-a1.na1dc1us.xxxxx.com:hystrix-xxxxxSearchService-8:2A08ED314C.200410163355836.895] [xreqid=3edbe2a0-bcdd-45fd-8251-af988f8d09ba] 
XXXXHttpProxy.handle:107) Error proxying to https://xxxxSearchService-na1us.micro.xxxx.com [java.net.SocketException: Connection reset]
java.net.SocketException: Connection reset
...
```
## Analysis
The microservice does not have any error during this time range, and this failed request actually was not found from microservice log yet.
We want to narrow down this exception. We know the monolithic will send request to microservice through the AWS load balancer.
Since the NLB was one of the new technologies we had introduced to this infrastructure, so we decided to check the NLB first.
Base on the Cloudwatch metrics, we found the load balancer reset count metric was higher than we expected it to be. 
So, we suspected the resets were directly causing the 503s in the environment. 
At the same time, we also checked the log from production environment, but we did not see such socket reset exception yet.
The ELB was used in production environment, we decided to run a quick test, we swap the NLB with ELB for the same Target Group.
Soon after the swap, we found the problem went away! Why?? We'd like to determine why we saw problems with the NLB that we did not see with the ELB. 
From the monolithic log, we picked the failed request from a specified monolithic server to a specified microservice to 
determine the time, then we searched the preview requests from the same monolithic server to the same microservice.
We found the previous request was sent about 8 minutes ago.
From AWS NLB documentation, we found that the NLB documented timeout behavior as below.
```
Fr each request that a client makes through a Network Load Balancer, the state of that connection is tracked. The connection is terminated by the target. 
If no data is sent through the connection by either the client or target for longer than the idle timeout, the connection is closed. 
If a client sends data after the idle timeout period elapses, it receives a TCP RST packet to indicate that the connection is no longer valid.
```
In other words, AWS NLB silently terminates the connection upon idle timeout. 
If an application tries to send data on the socket after idle timeout, it receives a TCP RST packet.
However, the ELB does not have such problem. In order to find out the difference between ELB and NLB, we decided to simulate the idle timeout, and captured the network traffics to compare.
### ELB idle timeout behaviour
THe default idle timeout of ELB is 60 seconds, and this setting can be changed. We just use default setting here.
When the connections expire through idle timeout, ELB sends a FIN packet to each connected party. As a result, the application can close the socket accordingly.
### NLB idle timeout behaviour
NLB has an idle timeout of 350 seconds which cannot be changed. 
When connections expire through idle timeout, NLB terminates the connections silently. 
An application that was not aware of this timeout, and it continued attempting to send data to the same socket. 
At that point, the NLB would notify the application that the connection has been terminated by sending it an RST packet.
Base on the traffic analysis, it can help us to answer why the "java.net.SocketException: Connection reset" was thrown.
Until now, we figured out why this issue occurred with NLB, because of NLB killed the connection when idle timeout occurs silently, and application continued using the same connection to send data. 
## Solution
If we want to use NLB, then we should consider enabling the TCP keep alive, use the OS builtin TPC keep alive mechanism to avoid NLB idle timeout.
We should make sure set TPC keep alive value well under 350 seconds. 
In Linux, the default /proc/sys/net/ipv4/tcp_keepalive_time is 7200 (2 hours), we can update this value like 60 seconds(which depend on services.)
After making this change, we found the java socket reset exception was gone.
## References
Tenable blog
