## Overview
Recently, we see another kind of connection reset exception occurred from the Istio service mesh.
We upgraded the Istio cluster to version 1.6.8, this kind of connection reset happened intermittently 
when a service called another service in the same mesh.  
This page discusses what's the root cause of this kind of connection reset exception, and how to resolve it.
## Problem
The microservices run on top of Istio service mesh. There was an example of "java.net.SocketException: Connection reset" stack trace 
when a service called another service via REST API. This connection reset caused the service to be unavailable.   
```
{ [-]
level: ERROR
logger_name: c.a.s.p.e.CustomExceptionHandler
message: getMessageWithRootCause >> Exception occurred.
process: NA
requestId: a15ebb00-a491-9f4c-8c30-efdffdaa54e6
serviceArchPath: NA
stack_trace: com.xxx.xxx.xxxxxx.exception.xxxxException: Required Upstream Service unreachable
at xxx.xxx.xxxxxx.exception.xxxxException.createError(PxxxxxerException.java:53)
....
at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1040)
at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:943)
at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006)
at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:909)
at javax.servlet.http.HttpServlet.service(HttpServlet.java:660)
at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883)
at javax.servlet.http.HttpServlet.service(HttpServlet.java:741)
...
at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74)
at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:343)
at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:373)
at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65)
at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:868)
at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1590)
at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(Unknown Source)
at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
at java.base/java.lang.Thread.run(Unknown Source)
Caused by: xxx.xxx.xxx.resiliency.exception.xxxCommandRetryException: Failed while retrying. java.net.SocketException: Connection reset
at xxx.xxx.xxx.resiliency.xxxCommand.throwIfNotRetryable(xxCommand.java:204)
...
at com.netflix.hystrix.HystrixCommand$2.call(HystrixCommand.java:302)
at com.netflix.hystrix.HystrixCommand$2.call(HystrixCommand.java:298)
...
at com.netflix.hystrix.strategy.concurrency.HystrixContexSchedulerAction$1.call(HystrixContexSchedulerAction.java:56)
at com.netflix.hystrix.strategy.concurrency.HystrixContexSchedulerAction$1.call(HystrixContexSchedulerAction.java:47)
...
at com.netflix.hystrix.strategy.concurrency.HystrixContexSchedulerAction.call(HystrixContexSchedulerAction.java:69)
at rx.internal.schedulers.ScheduledAction.run(ScheduledAction.java:55)
at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Unknown Source)
at java.base/java.util.concurrent.FutureTask.run(Unknown Source)
at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(Unknown Source)
at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
... 1 common frames omitted
Caused by: javax.ws.rs.ProcessingException: java.net.SocketException: Connection reset
at org.glassfish.jersey.client.internal.HttpUrlConnector.apply(HttpUrlConnector.java:284)
at org.glassfish.jersey.client.ClientRuntime.invoke(ClientRuntime.java:278)
at org.glassfish.jersey.client.JerseyInvocation.lambda$invoke$0(JerseyInvocation.java:753)
at org.glassfish.jersey.internal.Errors.process(Errors.java:316)
at org.glassfish.jersey.internal.Errors.process(Errors.java:298)
at org.glassfish.jersey.internal.Errors.process(Errors.java:229)
at org.glassfish.jersey.process.internal.RequestScope.runInScope(RequestScope.java:414)
at org.glassfish.jersey.client.JerseyInvocation.invoke(JerseyInvocation.java:752)
at org.glassfish.jersey.client.JerseyInvocation$Builder.method(JerseyInvocation.java:419)
at org.glassfish.jersey.client.JerseyInvocation$Builder.get(JerseyInvocation.java:319)
... 32 common frames omitted
Caused by: java.net.SocketException: Connection reset
at java.base/java.net.SocketInputStream.read(Unknown Source)
at java.base/java.net.SocketInputStream.read(Unknown Source)
at java.base/java.io.BufferedInputStream.fill(Unknown Source)
at java.base/java.io.BufferedInputStream.read1(Unknown Source)
at java.base/java.io.BufferedInputStream.read(Unknown Source)
at java.base/sun.net.www.http.HttpClient.parseHTTPHeader(Unknown Source)
at java.base/sun.net.www.http.HttpClient.parseHTTP(Unknown Source)
at java.base/sun.net.www.http.HttpClient.parseHTTP(Unknown Source)
at java.base/sun.net.www.protocol.http.HttpURLConnection.getInputStream0(Unknown Source)
at java.base/sun.net.www.protocol.http.HttpURLConnection.getInputStream(Unknown Source)
at java.base/java.net.HttpURLConnection.getResponseCode(Unknown Source)
at org.glassfish.jersey.client.internal.HttpUrlConnector._apply(HttpUrlConnector.java:390)
at org.glassfish.jersey.client.internal.HttpUrlConnector.apply(HttpUrlConnector.java:282)
... 43 common frames omitted

thread_name: http-nio-8080-exec-5
timestamp: 2020-10-23T00:26:14.871Z
x-correlation-id: a15ebb00-a491-4f4c-8c30-efdffdaa54e6-1
}
```
Below Splunk query showed how many "connection reset" exception occurred per day per service in a specified time range.
![connection-reset-istio-3](/images/connection-reset-istio-3.png)

## Analysis
Before jumping to analyze the root cause, let's take a look the what's the current service mesh architecture.
![istio-service-mesh](/images/IstioServiceMesh.png)
Service A will call Service B and Service C via HTTP inside the mesh, and the connection reset exception occurs 
when Service A calls Service B or Service C. The entire service mesh already enables mTLS by default from service to service.
In order to figure out why this kind of "Connection reset" exception occurs, we decide to enable the Istio sidecar debug log.
1. After enabling Istio sidecar debug log, there was an error message which indicated the connection was closed due to SSL handshake error.
```
2020-10-23T00:26:14.869776Z debug envoy filter [external/envoy/source/extensions/filters/listener/http_inspector/http_inspector.cc:37] http inspector: new connection accepted
2020-10-23T00:26:14.869797Z debug envoy conn_handler [external/envoy/source/server/connection_handler_impl.cc:411] [C3656911] new connection
2020-10-23T00:26:14.869803Z debug envoy connection [external/envoy/source/extensions/transport_sockets/tls/ssl_socket.cc:198] [C3656911] handshake error: 1
2020-10-23T00:26:14.869806Z debug envoy connection [external/envoy/source/extensions/transport_sockets/tls/ssl_socket.cc:226] [C3656911] TLS error: 268435612:SSL routines:OPENSSL_internal:HTTP_REQUEST
2020-10-23T00:26:14.869808Z debug envoy connection [external/envoy/source/common/network/connection_impl.cc:200] [C3656911] closing socket: 0
2020-10-23T00:26:14.869824Z debug envoy conn_handler [external/envoy/source/server/connection_handler_impl.cc:111] [C3656911] adding to cleanup list
```
2. Base on sidecar debug log, we confirmed the mTLS "STRICT" mode was enabled, and we also confirmed both the
Istio Authentication policy and Authorization policy were enabled correctly.

3. We also checked the log of a successful request from the sidecar B, we tried to find out is there any different. 
```
2020-10-24T17:37:23.593519Z debug envoy filter [external/envoy/source/extensions/filters/listener/original_dst/original_dst.cc:18] original_dst: New connection accepted
2020-10-24T17:37:23.593538Z debug envoy filter [external/envoy/source/extensions/filters/listener/tls_inspector/tls_inspector.cc:78] tls inspector: new connection accepted
2020-10-24T17:37:23.593562Z debug envoy filter [external/envoy/source/extensions/filters/listener/tls_inspector/tls_inspector.cc:148] tls:onServerName(), requestedServerName: outbound_.80_._.xxxx-service-b.preview.svc.cluster.local
2020-10-24T17:37:23.593570Z debug envoy filter [external/envoy/source/extensions/filters/listener/http_inspector/http_inspector.cc:37] http inspector: new connection accepted
2020-10-24T17:37:23.593612Z debug envoy conn_handler [external/envoy/source/server/connection_handler_impl.cc:411] [C3849060] new connection
2020-10-24T17:37:23.594383Z debug envoy connection [external/envoy/source/extensions/transport_sockets/tls/ssl_socket.cc:191] [C3849060] handshake expecting read
2020-10-24T17:37:23.594391Z debug envoy connection [external/envoy/source/extensions/transport_sockets/tls/ssl_socket.cc:191] [C3849060] handshake expecting read
2020-10-24T17:37:23.596346Z debug envoy connection [external/envoy/source/extensions/transport_sockets/tls/ssl_socket.cc:176] [C3849060] handshake complete``` 
```
After comparing the two log set between a successful request and a failed request, we found there was a missed log entry from the failed request.
```
2020-10-24T17:37:23.593562Z debug envoy filter [external/envoy/source/extensions/filters/listener/tls_inspector/tls_inspector.cc:148] tls:onServerName(), requestedServerName: outbound_.80_._.xxxx-service-b.preview.svc.cluster.local
``` 
The service A sidecar should send the outbound request via the "outbound_.80_._.xxxx-service-b.default.svc.cluster.local" router to sidecar of service B, but it did not, which it mean this specified outbound request was went through via the default PassthroughCluster, 
and the default PassthroughCluster required TLS traffic per the global istio mesh TLS mode "STRICT". This was the reason to cause connection reset issue. We think this was a mTLS related issue from Istio.

4. This is an issue of istio since 1.6x release, please refer below istio issue for more detail
```
https://github.com/istio/istio/issues/24379
```
There was a known issue of Istio 1.6.x release which complained about unexplained “Passthrough” was involved.
  
## Solution
As the root cause was already identified, the solution was also easy to apply. 
Increasing the default protocol sniffing timeout (meshConfig.protocolDetectionTimeout) to 5 seconds should resolve this istio related issue.
The default connectionTimeout value was 100 milli seconds. 
I run below command to apply the protocolDetectionTimeout for the entire Istio service mash.
```
istioctl manifest apply --set meshConfig.protocolDetectionTimeout=5s
```
I run below commands to double check the new protocolDetectionTimeout was applied to 5 seconds or not.
```
kubectl get configmap -n istio-system -o jsonpath={.data.mesh} istio
```
After applying the fix on production on Oct 26 evening, the connection reset exception was gone.
Attached below splunk query for a reference. This graph showed how many "connection reset" exception occurred per day per service in a specified time range.

![connection-reset-istio-4](/images/connection-reset-istio-4.png)

## Donate
如果本仓库对你有帮助，可以请作者喝杯速溶咖啡

![wechat_pay](/images/WeChatPay_2.jpeg)