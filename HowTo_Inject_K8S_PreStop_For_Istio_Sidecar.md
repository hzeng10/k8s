## Overview
In k8s pod graceful shutdown scenario, both Istio sidecar(Envoy proxy) container and application container receives the SIGTERM at same time. By default, the graceful termination period is 5 seconds for the Envoy proxy,  and application container graceful termination period is usually longer than 5 seconds. So there is a common issue that the sidecar container is terminated early than the application container, if the application is still doing anything as part of its shutdown process, and attempts to make an outbound connection AFTER Istio sidecar has shut down, it'll fail because the IPTables rules are sending traffic to a non-existent envoy proxy. K8s plans to offer a KEP feature to overcome such problem, but this feature is still not ready yet, for more detail please refer KEP 753. It means we need a workaround to resolve this common problem.

As we already know that k8s pod offers a preStop hook which can help containers to do some processes during graceful shutdown. So, in order to resolve this issue, we can also think about using this preStop hook for sidecar, as preStop hook can help to delay shutting down the envoy, so that the application containers can still get a chance to complete the outbound connection during graceful shutdown.

There is a challenge, the sidecar is injected automatically when deploying the application in k8s cluster. How can we add a preStop hook for the Isito sidecar? Istio offers a template YMAL file which includes the envoy proxy sidecar injection deployment, the template YMAL will be persistent as k8s ConfigMap object. This template yaml file can be found from Istio path ..\install\kubernetes\istio-demo.yaml. The idea will edit the template ConfigMap, and inject the preStop hook for envoy proxy, save this change, so that the automatically injected side car will include the preStop accordingly. This page will share the detail about this idea, includes the installation steps, what's kind of setting to change, and how to verify the workaround.
## Istio Installation
Istio 1.5 introduces some new features and the installation process is also changed.
### Download
For macOS/linux, please run below command to download Isito release.
```
$ curl -L https://istio.io/downloadIstio | sh -
```
### Install
Istio offers several installation configuration profiles, please refer Installation Configuration Profiles for detail. For testing purpose, we recommend to use demo profile, as demo profile includes all Istio features.
1. Run below command to install Istio with demo profile.
```
$ istioctl manifest apply --set profile=demo
 
Detected that your cluster does not support third party JWT authentication. Falling back to less secure first party JWT
- Applying manifest for component Base...
✔ Finished applying manifest for component Base.
- Applying manifest for component Pilot...
✔ Finished applying manifest for component Pilot.
Waiting for resources to become ready...
- Applying manifest for component EgressGateways...
- Applying manifest for component IngressGateways...
- Applying manifest for component AddonComponents...
✔ Finished applying manifest for component EgressGateways.
✔ Finished applying manifest for component IngressGateways.
✔ Finished applying manifest for component AddonComponents.
 
✔ Installation complete
```
2. Add a namespace label to instruct Istio to automatically inject Envoy sidecar proxies when you deploy your application later.
```
$ kubectl label namespace default istio-injection=enabled
 
namespace/default labeled
```
## Inject PreStop into template yaml directly
As mentioned early, the sidecar injection template yaml is presented as  k8s ConfigMap object, the name of this ConfigMap object is "istio-sidecar-injector". So we can edit the content of ConfigMap, add preStop hook for the injected envoy proxy. Please follow below steps:
1. Use kubectl to edit configmap
```
$ kubectl -n istio-system edit configmap istio-sidecar-injector
```
2. Search of "lifecycle" from the vi windows, the injected envoy proxy container will include lifecycle tag if the configmap object has the value (values.global.proxy.lifecycle).
```yaml
{{- if .Values.global.proxy.lifecycle }}
  lifecycle:
    {{ toYaml .Values.global.proxy.lifecycle | indent 4 }}
{{- end }}
```
3. In this ConfigMap object, the data filed includes "config" and "values" sub fields, we can add the preStop hook content into the values filed accordingly.  Search of "proxy": , and add below content into proxy {} json body. For more detail, please refer below screenshots.
```yaml
"lifecycle": {
   "preStop": {
      "exec": {
         "command": ["/bin/bash", "-c", "while [ $(netstat -plunt | grep tcp | grep -v envoy | wc -l | xargs) -ne 0 ]; do sleep {{ .Spec.TerminationGracePeriodSeconds }}; done"]
       }
    }
 },
```
![lifecyle_prestop](/images/lifecycle_prestop.png)

4. Save this change to refresh the updated ConfigMap into k8s cluster.
## Inject PreStop into template yaml by using annotation
Inject the preStop into template yaml directly will require to apply the preStop for the entire k8s cluster.
There is another option to inject the preStop by using annotation per each deployment. Each deployment can have its own preStop hook for the injected sidecar (Envoy proxy) or does not require it neither.
In order to inject preStop for each deployment, we can take below steps:
1. Use kubectl to edit configmap
```
$ kubectl -n istio-system edit configmap istio-sidecar-injector
```
2. Search of "lifecycle" from the vi windows, make changes as below. 
The injected sidecar template will retrieve the 'sidecar.istio.io/preStopCommand' annotation from application deployment if annotation presents.
```yaml
{{- if .Values.global.proxy.lifecycle }}
  lifecycle:
    {{ toYaml .Values.global.proxy.lifecycle | indent 4 }}
{{- else if isset .ObjectMeta.Annotations `sidecar.istio.io/preStopCommand` }}
  lifecycle:
    preStop:
      exec:
        command: ["/bin/bash", "-c", "{{ annotation .ObjectMeta `sidecar.istio.io/preStopCommand` `sleep 30` }}"]
{{- end }}
``` 
3. Update the k8s application deployment to include the preStop annotation:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  selector:
    matchLabels:
      app: myapp
  replicas: 1
  template:
    metadata:
      # add the preStop annotation for the injected sidecar  
      annotations:
        sidecar.istio.io/preStopCommand: "while [ $(netstat -plunt | grep tcp | grep -v envoy | wc -l | xargs) -ne 0 ]; do sleep 30; done"
      labels:
        app: myapp
    spec:
      terminationGracePeriodSeconds: 32
      containers:
      - name: myapp
        image: hzeng10-MacBook-Pro.local:5000/myapp:v1
        imagePullPolicy: Always
        resources:
          requests:
            cpu: 0.5
            memory: 256Mi
          limits:
            cpu: 0.75
            memory: 300Mi
        ports:
        - containerPort: 8080
        lifecycle:
            preStop:
              exec:
                command: ["/bin/bash", "-c", "sleep 30"]
---
```

## Deploy k8s application to verify the injected preStop
1. Use kubectl apply command to deploy application into k8s cluster
```
$ kubectl apply -f ~/dev/k8s-deployment/<app name>/deployment.yaml
```
2. Use kubectl get pod command check the injected envoy container deployment json.
```
$ kubectl get pod <app pod name> -o=json
```
3. Use kubectl log command to printout the injected envoy proxy log to console.
```
$ kubectl logs -f -c istio-proxy <app pod name>
```
4. Use kubectl delete pod to delete the pod and check the log output from injected envoy proxy in step 3. The injected sidecar will run the preStop hook (sleep for several seconds) to delay the shutdown.
```
$ kubectl delete pod <app pod name>
pod "<app pod name>" deleted
```
Envoy proxy termination log: 
```
[2020-04-16T01:56:27.440Z] "- - -" 0 - "-" "-" 2232 7039 10813 - "-" "-" "-" "-" "52.21.95.189:443" PassthroughCluster 172.17.0.13:57980 52.21.95.189:443 172.17.0.13:57978 - -
2020-04-16T01:58:23.257099Z info    Agent draining Proxy
2020-04-16T01:58:23.257282Z info    Status server has successfully terminated
2020-04-16T01:58:23.257343Z error   accept tcp [::]:15020: use of closed network connection
2020-04-16T01:58:23.257420Z info    watchFileEvents has successfully terminated
2020-04-16T01:58:23.257458Z info    Watcher has successfully terminated
2020-04-16T01:58:23.263318Z info    Graceful termination period is 5s, starting...
[Envoy (Epoch 0)] [2020-04-16 01:58:25.271][17][warning][main] [external/envoy/source/server/server.cc:513] caught SIGTERM
[Envoy (Epoch 0)] [2020-04-16 01:58:25.271][17][warning][main] [external/envoy/source/server/server.cc:513] caught SIGTERM
```
## Donate
如果本仓库对你有帮助，可以请作者喝杯速溶咖啡

![wechat_pay](/images/WeChatPay_2.jpeg) 

 




