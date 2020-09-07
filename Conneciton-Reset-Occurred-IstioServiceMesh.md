## Overview
Recently, there were another high number of connection reset exceptions occurred from Istio service mesh, as a result the microservice 
authentication validation was failed. And this issue impacted most of services on production.
This page discusses what's the root cause of this kind of connection reset exception, and how to resolve it.
## Problem
The microservices run on top of Istio service mesh. Many authentication validation error messages were reported to dashboard,
those errors were caused by a connection reset exception when service sent validation requests to the identity management service, and the entire service calls were failed. 
```
@timestamp: 2020-05-29T19:05:44.614+00:00
   @version: 1
   level: ERROR
   level_value: 40000
   logger_name: com.xxx.xxx.xxx.xxx.xxx.xxxServiceAuthenticationHelper
   message: Problem validating service token
   stack_trace: com.xxx.xxx.xxx.xxx.exception.xxxConnectorRuntimeException: Problem processing validate xxx token response
	at com.xxx.xxx.xxx.xxx.impl.xxxConnectorImpl.validateToken(xxxConnectorImpl.java:535)
	at com.xxx.xxx.xxx.xxx.impl.xxxConnectorImpl.validateAccessToken(xxxConnectorImpl.java:321)
    ...
Caused by: java.net.SocketException: Connection reset
	at java.net.SocketInputStream.read(SocketInputStream.java:210)
	at java.net.SocketInputStream.read(SocketInputStream.java:141)
	at sun.security.ssl.InputRecord.readFully(InputRecord.java:465)
	at sun.security.ssl.InputRecord.read(InputRecord.java:503)
	at sun.security.ssl.SSLSocketImpl.readRecord(SSLSocketImpl.java:975)
	at sun.security.ssl.SSLSocketImpl.performInitialHandshake(SSLSocketImpl.java:1367)
	at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1395)
	at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1379)
	at sun.net.www.protocol.https.HttpsClient.afterConnect(HttpsClient.java:559)
	at sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.connect(AbstractDelegateHttpsURLConnection.java:185)
	at sun.net.www.protocol.http.HttpURLConnection.getOutputStream0(HttpURLConnection.java:1334)
	at sun.net.www.protocol.http.HttpURLConnection.getOutputStream(HttpURLConnection.java:1309)
	at sun.net.www.protocol.https.HttpsURLConnectionImpl.getOutputStream(HttpsURLConnectionImpl.java:259)
    ...
```
Below Splunk query showed how many "connection reset" exception occurred per day per service in a specified time range.
![connection-reset-istio-1](/images/connection-reset-istio-1.png)

## Analysis
After analyzed the service logs, we saw a common pattern, the exception was thrown after 1 second when service sent the outbound request
to identity management service. At that time, I checked the identity management service, this service was reachable, and it was also available.
In order to figure out what's going from Isito service mesh, I enabled the debug log from Istio sidecar (envoy proxy) from service mesh.
```
istioctl proxy-config log xxxx-85bb5446f7-g5hqn --level=debug
```
After reproducing the same issue, I got below Envoy debug log which indicated the outbound connection was failed due to connect timeout after 1 second.
It means the sidecar envoy uses a 1 second connect timeout for all outbound connections.
```
Sidecar envoy proxy debug log:
[Envoy (Epoch 0)] [2020-05-29 22:04:05.063][27][debug][filter] [external/envoy/source/extensions/filters/listener/original_dst/original_dst.cc:18] original_dst: New connection accepted
[Envoy (Epoch 0)] [2020-05-29 22:04:05.063][27][debug][filter] [external/envoy/source/extensions/filters/listener/tls_inspector/tls_inspector.cc:78] tls inspector: new connection accepted
[Envoy (Epoch 0)] [2020-05-29 22:04:05.066][27][debug][filter] [external/envoy/source/extensions/filters/listener/tls_inspector/tls_inspector.cc:148] tls:onServerName(), requestedServerName: cc-api-xxx-xxx.xxx.io
[Envoy (Epoch 0)] [2020-05-29 22:04:05.066][27][debug][filter] [external/envoy/source/common/tcp_proxy/tcp_proxy.cc:232] [C331] new tcp proxy session
[Envoy (Epoch 0)] [2020-05-29 22:04:05.066][27][debug][filter] [external/envoy/source/common/tcp_proxy/tcp_proxy.cc:381] [C331] Creating connection to cluster outbound|443||cc-api-xxx-xxx.xxx.io
[Envoy (Epoch 0)] [2020-05-29 22:04:05.066][27][debug][pool] [external/envoy/source/common/tcp/conn_pool.cc:83] creating a new connection
[Envoy (Epoch 0)] [2020-05-29 22:04:05.066][27][debug][pool] [external/envoy/source/common/tcp/conn_pool.cc:364] [C332] connecting
[Envoy (Epoch 0)] [2020-05-29 22:04:05.066][27][debug][connection] [external/envoy/source/common/network/connection_impl.cc:698] [C332] connecting to 52.xx.xx.xx:443
[Envoy (Epoch 0)] [2020-05-29 22:04:05.067][27][debug][connection] [external/envoy/source/common/network/connection_impl.cc:707] [C332] connection in progress
[Envoy (Epoch 0)] [2020-05-29 22:04:05.067][27][debug][pool] [external/envoy/source/common/tcp/conn_pool.cc:109] queueing request due to no available connections
[Envoy (Epoch 0)] [2020-05-29 22:04:05.067][27][debug][conn_handler] [external/envoy/source/server/connection_handler_impl.cc:353] [C331] new connection
[Envoy (Epoch 0)] [2020-05-29 22:04:06.066][27][debug][pool] [external/envoy/source/common/tcp/conn_pool.cc:403] [C332] connect timeout
[Envoy (Epoch 0)] [2020-05-29 22:04:06.066][27][debug][connection] [external/envoy/source/common/network/connection_impl.cc:101] [C332] closing data_to_write=0 type=1
[Envoy (Epoch 0)] [2020-05-29 22:04:06.066][27][debug][connection] [external/envoy/source/common/network/connection_impl.cc:192] [C332] closing socket: 1
[Envoy (Epoch 0)] [2020-05-29 22:04:06.067][27][debug][pool] [external/envoy/source/common/tcp/conn_pool.cc:124] [C332] client disconnected
[Envoy (Epoch 0)] [2020-05-29 22:04:06.067][27][debug][filter] [external/envoy/source/common/tcp_proxy/tcp_proxy.cc:484] [C331] connect timeout
[Envoy (Epoch 0)] [2020-05-29 22:04:06.067][27][debug][filter] [external/envoy/source/common/tcp_proxy/tcp_proxy.cc:381] [C331] Creating connection to cluster outbound|443||cc-api-xxx-xxx.xxx.io
[Envoy (Epoch 0)] [2020-05-29 22:04:06.067][27][debug][connection] [external/envoy/source/common/network/connection_impl.cc:101] [C331] closing data_to_write=0 type=1
[Envoy (Epoch 0)] [2020-05-29 22:04:06.067][27][debug][connection] [external/envoy/source/common/network/connection_impl.cc:192] [C331] closing socket: 1
[Envoy (Epoch 0)] [2020-05-29 22:04:06.067][27][debug][conn_handler] [external/envoy/source/server/connection_handler_impl.cc:86] [C331] adding to cleanup list
[Envoy (Epoch 0)] [2020-05-29 22:04:06.067][27][debug][pool] [external/envoy/source/common/tcp/conn_pool.cc:238] [C332] connection destroyed

[2020-05-29T22:04:05.066Z] "- - " 0 UF,URX "" "" 0 0 1000 - "" "" "" "-" "52.xx.xx.xx:443" outbound|443||cc-api-xxx-xxx.xxx.io - 52.xx.xx.xx:443 172.17.0.13:45202 - -
```
Base on above debug log, I continued to check the sidecar envoy proxy configuration via below command. This command can help to dump the cluster setting.
```
istioctl proxy-config cluster xxxx-85bb5446f7-g5hqn --port 443 -o json
[
{
"name": "outbound|443||cc-api-xxx-xxx.xxx.io",
"type": "STRICT_DNS",
"connectTimeout": "1s",
"loadAssignment": {
"clusterName": "outbound|443||cc-api-xxx-xxx.xxx.io",
....
```
After reviewing the config dump, I confirmed there was a 1 second connectTimeout for the outbound connections.
The debug log and envoy config dump already explained the root cause, there was a 1 second connectTimeout setting for envoy outbound connection, 
so envoy got a connection failure if the remote service did not accept the connection in 1 second. This could happen frequently due to network latency.
 
## Solution
As the root cause was already identified, the solution was also easy to apply. 
The default connectionTimeout value from Istio was 10 seconds. Increasing the  connectTimeout from 1 second to 10 second can help to resolve this issue.
I run below command to apply the connectTimeout for the entire Istio service mash.
```
istioctl manifest apply --set meshConfig.connectTimeout=10s
```
I run below commands to double check the new connectTimeout was applied successfully or not.
```
istioctl proxy-config cluster xxxx-85bb5446f7-g5hqn --port 443 -o json
```
After applying the fix on production on July 8 evening, the connection reset exception was gone, and all the authentication validation calls were succeed.
Attached below splunk query for a reference. This graph showed how many "connection reset" exception occurred per day per service in a specified time range.

![connection-reset-istio-2](/images/connection-reset-istio-2.png)

## Donate
如果本仓库对你有帮助，可以请作者喝杯速溶咖啡

![wechat_pay](/images/WeChatPay_2.jpeg)