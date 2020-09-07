## Overview
Recently, there was another high number of HTTP 503 UC/UF/UH errors occurred from one specified service on production, 
as a result this service returned many HTTP 503 errors to the API gateway.
This page discusses what's the root cause of this kind of 503 errors, and how to resolve it.
## Problem
The microservice run on top of k8s + Istio service mesh. The API client sent HTTP requests via several hops to service.
Attached below request flow for a reference. 
```
APIClient ---> MonolithicAPIGatway ---> AWS LB ---> IstioIngressGateWay ---> IstioSidecar ---> Service
```
From the splunk log, there were many 503 UC/UF from the Istio Sidecar logs, and there were also many 503 UH errors from
the Istio IngressGateway logs, as a result those 503 UH errors were returned to Monolithic API gateway which were sent back to API client.
Attached below HTTP 503 errors as examples
```
HTTP 503 UC error from Istio sidecar envoy
[2020-08-20T11:54:22.063Z] "GET /xxxxx/xxxxx/3BACBLblqZhBbNT7HViM8fgRs0czKdmYEH-wWUzLIbwEHN7LCEly0lLT_KRijIUDvgPQLisMN-j1UgPNRMbwJq0N_QwhR_Fpu HTTP/1.1" 503 UC "-" "-" 0 95 1213 - "10.71.65.158" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) AcrobatServices/20.12.20043 Chrome/80.0.0.0 Safari/537.36" "6cab429c-5085-4091-a714-c30ae48fe621" "xxxxx-xxx.micro.xxxxx.com" "127.0.0.1:8080" inbound|80|http|xxxxx.prod.svc.cluster.local 127.0.0.1:50246 100.64.15.41:8080 10.71.65.158:0 outbound_.80_._.xxxxx.prod.svc.cluster.local default

HTTP 503 UF error from Istio sidecar envoy
[2020-08-20T13:36:24.347Z] "GET /xxxxx/xxxxx/3CAEBLblqQhADTNs4k5NsKzSWDCFwBgNWqP8qiRcvPB3sAzGd32Fg9czGHIA1wRYmRdM1vjut7XaObydDhO3Hf0RldDT_OW50 HTTP/1.1" 503 UF "-" "-" 0 91 0 - "10.69.102.12" "Mozilla/5.0 (Windows NT 10.0; WOW64; Trident/7.0; rv:11.0) like Gecko" "ad6fb381-691e-44d8-a2d5-ff88fc84f8d4" "xxxxx-xxx.micro.xxxxx.com" "127.0.0.1:8080" inbound|80|http|xxxxx.prod.svc.cluster.local - 100.64.11.157:8080 10.69.102.12:0 outbound_.80_._.xxxxx.prod.svc.cluster.local default

HTTP 503 UH error from Istio Ingress gateway
[2020-08-20T23:21:25.233Z] "GET /xxxxx/xxxxx/3QAFBLblqDhC59dcYAXUsMgv_mBpCtarUeIvIwJn7C6MtsKhepyxJ_bVgrqcPZOMi1DnJIIzmZkYFu1yjo7vyTtJuW509JFAx HTTP/1.1" 503 UH "-" "-" 0 19 0 - "10.70.38.234" "Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.135 Safari/537.36" "f2bf2de3-4a01-4cef-b9f7-803086033014" "xxxxx-xxx.micro.xxxxx.com" "-" - - 100.64.3.159:443 10.70.38.234:25117 xxxxx-xxx.micro.xxxxx.com -
```
Below Splunk query showed how many "503 U*" errors occurred per day per service in a specified time range.
This specified service with field* prefix had much more 503 errors than other services at the same time.
![503-Error-Per-Service-Per-Day-1](/images/503-Error-Per-Service-Per-Day-1.png)

## Analysis
After analyzed the service logs, we saw a common pattern, those 503 errors occurred very frequently, and those 503 errors were not from the service application itself.
There was not any application related error or exception when the 503 error occurred. The istio sidecar (envoy) reported those error directly.
The UC/UF/UH error codes came from Istio envoy which indicated the upstream connection failure or upstream unhealthy
```
UC: Upstream connection termination
UF: Upstream connection failure
UH: no healthy upstream hosts
```
Base on those pattern and error codes, I suspected the service was not reachable for some reasons. There were also many tomcat startup event from the service logs.
The startup events indicated the service was restarted very frequently. And the startup events occurred around 503 errors, which showed a strong evidence.
Those 503 errors occurred due to the service was restarted, so Istio sidecar cannot connect to the service.
The next question is here? Why the service was restarted frequently? When the tomcat was restarted, there was not any application error or exception yet.
Is it possible that the k8s kills the service container? In order to figure out why this restart happens, I run below command to collect more detail event from k8s pods. 
```
kubectl describe pod xxxx-7ddcc7745f-lh7lm
```
This comment collects all pod events/activities. From the output, I saw the container was killed by k8s due to OOM, and it was restarted for more 15 times. 
Attached below output for a reference.

![k8s-OOMKilled-servie](/images/k8s-OOMKilled-Service.png)

Base on above k8s event, I figured out why the service was restarted frequently, the service container consumed more memory than the memory limitation 3G, 
so k8s killed this container, and restarted the container. As a result, those 503 errors occurred during before the container was ready to service.
 
## Solution
As the root cause was already identified, the solution was also easy to apply. It looks like the container may need 
consuming more memory. So, increase the k8s memory limitation from 3G to 4G from deployment, which can be quick workaround here.
```yaml
      containers:
      - name: servicename
        image: myregistry/xxxx/fieldxxxx
        resources:
          requests:
            cpu: 0.75
            memory: 4G
          limits:
            cpu: 1.5
            memory: 4G
```
After deploying the fix on production, it should keep monitoring for several days. Keep monitoring the service for several days which
can help to make sure there is not any potential memory leak from this service. 
Attached a new splunk graph to show how many 503 error per day in a specified time range. After applying the fix, the 503 error was decreased to 0,
which means this fix can help to resolve the issue.
 
![Service-503-per-day-with-fix](/images/Service-503-per-day-with-fix-2.png)

## Donate
如果本仓库对你有帮助，可以请作者喝杯速溶咖啡

![wechat_pay](/images/WeChatPay_2.jpeg)
