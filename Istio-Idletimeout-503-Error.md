## Overview
Recently, there were another high number of HTTP 503 UC errors occurred from Istio service mesh, as a result many microservices
cannot complete the requests. This page discusses what's the root cause of this kind of HTTP 503 error, and how to resolve it.
## Problem
The microservices run on top of Istio service mesh. The Istio Ingressgateway returned HTTP 503 error randomly, 
but there was not any application error or exception from the service log at the same time.
```
[2020-07-10T15:55:03.935Z] "POST /contacts HTTP/1.1" 503 UC "-" "-" 70 95 722 - "172.20.24.35" "Apache-HttpClient/4.5.2 (Java/1.8.0_60)" "APITEST_RELAY__rid_6762633084098940_eeaa4482-1747-564d-818c-1672b927047e" "xxxxx-xxx.micro.xxxxx.com" "172.20.24.56:8080" outbound|80||xxxxx.xxx.svc.cluster.local 172.20.24.95:48666 172.20.24.95:443 172.20.24.35:16515 xxxxx-xxx.micro.xxxxx.com -
```
Below Splunk query showed how many "503 UC" error occurred per day per service in a specified time range.

![503UC-error-per-service-per-day](/images/503UC-Error-Per-Service-Per-Day.png)

## Analysis
After analyzed the Istio ingress gateway logs, I saw a common pattern, the 503 UC error occurred after 60 seconds when previous request 
was sent to the service through the Istio sidecar. At that time, I checked the service which was reachable, and there was not any error or exception from
service log. The 503 UC error indicated the upstream connection termination, which means the connection may close by upstream.
Is it possible the Istio sidecar close the connection for some reason? I captured several requests from Ingress gateway log, all those requests were from a same source, and were sent to a same destination.
```
UC: Upstream connection termination
Source: "172.20.24.95:48666"
Destination: "172.20.24.56:8080" 

[2020-07-10T15:55:03.935Z] "POST /contacts HTTP/1.1" 503 UC "-" "-" 70 95 722 - "172.20.24.35" "Apache-HttpClient/4.5.2 (Java/1.8.0_60)" "APITEST_RELAY__rid_6762633084098940_eeaa4482-1747-564d-818c-1672b927047e" "xxxxx-xxx.micro.xxxxx.com" "172.20.24.56:8080" outbound|80||xxxxx.xxx.svc.cluster.local 172.20.24.95:48666 172.20.24.95:443 172.20.24.35:16515 xxxxx-xxx.micro.xxxxx.com -
[2020-07-10T15:53:58.644Z] "HEAD /contacts HTTP/1.1" 200 - "-" "-" 0 0 12 12 "172.20.24.4" "Apache-HttpClient/4.5.3 (Java/1.8.0_162)" "2f5f86bc-ee06-42a4-9a67-ee520cceda1c" "xxxxx-xxx.micro.xxxxx.com" "172.20.24.56:8080" outbound|80||xxxxx.xxx.svc.cluster.local 172.20.24.95:48666 172.20.24.95:443 172.20.24.4:3347 xxxxx-xxx.micro.xxxxx.com -
[2020-07-10T15:53:48.799Z] "POST /contacts/search HTTP/1.1" 200 - "-" "-" 122 162 15 15 "172.20.24.35" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36" "c1204e5a-472b-435d-9faf-0e613114b687" "xxxxx-xxx.micro.xxxxx.com" "172.20.24.56:8080" outbound|80||xxxxx.xxx.svc.cluster.local 172.20.24.95:48666 172.20.24.95:443 172.20.24.35:1720 xxxxx-xxx.micro.xxxxx.com -
[2020-07-10T15:53:43.665Z] "POST /contacts/search HTTP/1.1" 200 - "-" "-" 28 83 14 14 "172.20.24.35" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36" "5d9bcc8b-d6e4-48a8-912c-93ca615ba5a2" "xxxxx-xxx.micro.xxxxx.com" "172.20.24.56:8080" outbound|80||xxxxx.xxx.svc.cluster.local 172.20.24.95:48666 172.20.24.95:443 172.20.24.35:1720 xxxxx-xxx.micro.xxxxx.com -
...
```
From the above request activities, I found the interval between failed request and the previous request already exceed 60 seconds. 
I suspected the 60 seconds may an idle timeout value of the Istio sidecar. 
After checking the service pod deployment, I found there was a k8s ENV "ISTIO_META_IDLE_TIMEOUT" which set the value as 60 seconds.
This ENV var defined the idle timeout for downstream, in this context, the Isito Ingress gateway was the downstream of Istio sidecar.
```
kubectl get pod xxxxx-xxxx-d77649d4f-8lqtx -o yaml | grep -B1 60s                                                                                                                                                                                                                               
<aws:xxxx-xxx-xxxx>
    - name: ISTIO_META_IDLE_TIMEOUT
      value: 60s
```
An incorrect ENV variable "ISTIO_META_IDLE_TIMEOUT" was injected into the Istio sidecar, which enforced Istio sidecar to close
the downstream idle connection after 60 seconds, so the HTTP 503 UC error occurred from Istio Ingress gateway.   
 
## Solution
As the root cause was already identified, the solution was also easy to apply.
Removed the "ISTIO_META_IDLE_TIMEOUT" ENV from the Istio sidecar injection template, or change the value from 60 seconds as a large value like 3600s, and re-apply the k8s pod deployment.
After applying the fix on July 15 evening, the HTTP 503 UC error was gone, and those impacted services resume operation.
Attached below splunk query for a reference. This graph showed how many "503 UC" error occurred per day per service in a specified time range.

![503UC-Error-Per-Service-Per-Day-Fix](/images/503UC-Error-Per-Service-Per-Day-Fix.png)

## Donate
如果本仓库对你有帮助，可以请作者喝杯速溶咖啡

![wechat_pay](/images/WeChatPay_2.jpeg)