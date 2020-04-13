# Introduction:

A Docker registry is a place where we can store images i.e. docker push, and let third-parties get them i.e. docker pull. 
Docker Hub is the default registry. So, setting this local docker registry can help us to push/pull image with Kubernetes. 
Below steps will use zero-downtime-rollingupdate service as an example to build the development image.

### Precondition:
1. Install docker on Mac laptop. In this page, we assume this Mac laptop is the primary development machine.
2. Follow the steps from zero-downtime-rollingupdate repository to build the test docker image.
3. Add the Insecure registries entry from the Docker desktop of Mac machine. 
Please refer below screen shot for a reference. The below IP address "192.150.23.217" is the machine of another docker. We assume the second docker is installed on a Linux VM. 
If you prefer to use Mac OS as the registry, then the IP address will be the Mac's IP address accordingly.
![docker insure registry](/images/Docker-Insecure-Registry.png)
4. From the MiniKube machine, update the InsecureRegistry with the same Insecurity registry from the Mac machine. By default, the Docker will pull image via HTTPS. 
```
sudo vi ~/.minikube/machines/minikube/config.json
```
Find the "InsecureRegisty" filed, and update the correct private docker image registry, then restart MiniKube.
```
"InsecureRegistry": [
"192.150.23.217:5000"
],
```
Please refer below screenshots for reference.
![minikube_config](/images/docker-registry.png)

### Steps to setup docker registry
Docker registry is just a Docker image, so we can just start the local docker registry via a running container. 
Below are the steps to setting up a local docker registry.

#### Linux
1. Install a docker in a Ubuntu Linux VM by using command:
```
$ sudo apt install docker.io. 
```
Assume the IP of this linux VM is "192.150.23.217".

2. Assign permission to Ubuntu admin user(hzeng10) to avoid using "sudo" when launch docker command from Ubuntu Linux VM. Without those setting, it will get permission denied when trying to connect to the Docker daemon socket.
``` 
$ sudo usermod -aG docker hzeng10
$ sudo usermod -aG root hzeng10
$ sudo chmod 777 /var/run/docker.sock
```
3. Edit "/etc/docker/dameon.json" file from Ubuntu Linux VM to avoid docker push HTTP error. Docker uses "https" by default to connect docker registry, we can workaround this HTTP error by adding an "insecure registries" entry.
If this file does not exist, then create a new one, add below content into this file. IP "192.150.23.217" is the Ubuntu Linux VM which will host the docker registry.
```
{

  "insecure-registries" : ["192.150.23.217:5000"]

}
```
#### macOS

4. Run below command to launch a registry container with port 5000 from Ubuntu Linux VM or Mac machine.
```
$ docker run -d -p 5000:5000 --restart=always --name registry registry:2
```
then run 
```
docker ps
```
to check the registry container is running successfully.

5. Create image tag for "zero-downtime-rollingupdate-img" from machine, we use this image to push it to docker registry.
```
$ docker tag zero-downtime-rollingupdate-img 192.150.23.217:5000/zero-downtime-rollingupdate:v1
```
-  "zero-downtime-rollingupdate-img" is the image name of this application.
-  "192.150.23.217" is the ip address of docker registry machine. (The Ubuntu Linux VM or the Mac Machine)
6. Run below command to push development image to docker registry.
```
$ docker push 192.150.23.217:5000/zero-downtime-rollingupdate:v1
```
Verify the image can be pushed to docker registry successfully or not.
```
$ curl -X GET http://192.150.23.217:5000/v2/_catalog
```
and the expected response will like below:
```
{"repositories":["zero-downtime-rollingupdate"]}
```
After push the dev image to docker registry, we can also pull this image from Kubernetes deployment.yml file accordingly. 
Attached below setting as a reference.
```yaml
    spec:
      containers:
      - name: zero-downtime-rollingupdate
        image: 192.150.23.217:5000/zero-downtime-rollingupdate:v1
        imagePullPolicy: Always

