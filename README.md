# Jenkins ci-cd k8s Pipeline

In this guide we will use Jenkins to attain full ci-cd integration for our pipeline for a sample webpage app using k8s.

## 1. Create a node

Create a master node and configure the firewalls to pass in http traffic and jenkins on port ```8080```.

## 2. Install prerequisite plugins

Install java, wget, git, docker and kubernetes on the master node with the following command:

```sudo yum install -y java wget git docker kubectl```

## 3. Configure docker

Configure docker by placing root user into the docker group. This way, we will not have to use ```sudo``` every time we try to run a docker command. First, create the docker group:

```sudo groupadd docker```

Place the root user into the docker group:

```sudo usermod -aG docker $USER```

Restart the master node through GCP so new user permissions for the docker group take effect.

## 4. Install jenkins

Install jenkins (CentOS 7):

```
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum install -y jenkins
```

Place the jenkins user into the docker group. If this is not done then jenkins will not run docker commands as jenkins cannot run as sudo:

```sudo usermod -aG docker jenkins```

Restart the master node so new user permissions for the docker group take effect. Go into the master node and start the jenkins service:

```sudo service jenkins start```

Copy the master node's ip into the browser as ```<master node ip>:8080>``` and follow the instructions to setting up a user account for jenkins.

## 5. Setting up docker

Start the docker service:

```sudo service docker start```

Fork the above repository into your own Github account and clone:

```git clone https://github.com/<Your Github Account>/Jenkins_CD-CD_Kubernetes.git```

```cd``` into this repository and build the docker image and push to dockerhub:

```
docker build . -t hellowhale
docker tag hellowhale <Your Dockerhub Account>/hellowhale
docker login -u <Your Dockerhub Account> -p <Your Dockerhub Password>
docker push <Your Dockerhub Account>/hellowhale
```

## 6. Create k8s cluster for hellowhale deployment

Authenticate with google cloud and follow the instructions:

```gcloud auth login```

Create a cluster of size 1 with:

```gcloud container clusters create pipeline5 --zone europe-west2-c --num-nodes 1```

Create a deployment for the docker image:

```kubectl create deployment hellowhale --image ilyashusain/hellowhale```

Expose the deployment with ```--type LoadBalancer```:

```kubectl expose deployment/hellowhale --port 80 --name hellowhalesvc --type LoadBalancer```

Run ```kubectl get svc``` to see the ip address for the above LoadBalancer, enter it into the browser to see the webpage.

## 7. Configure jenkins in the console

Copy the ```/home/$USER/.kube/config``` file to ```/var/lib/jenkins/.kube```. If the ```/.kube``` directory in ```/var/lib/jenkins/.kube``` does not exist then create it. Use sudo when copying, as such:

```sudo cp /home/$USER/.kube/config /var/lib/jenkins/.kube/```

Change ownership of the above file so that jenkins owns it and make it executable, without doing so we cannot run ```kubectl``` commands:

```
sudo chown jenkins:jenkins /var/lib/jenkins/.kube/config
chmod +x /var/lib/jenkins/.kube/config
```

Restart the jenkins service:

```sudo service jenkins restart```

## 8. Configure jenkins in the browser

Once you have logged in to jenkins, navigate to ```Manage Jenkins > Configure System```. Under ```Global Properties``` tick environment variables and put ```DOCKER_HUB``` under ```Name``` and your dockerhub password under ```Value```.

Under ```Jenkins Location``` in the ```Jenkins URL``` put ```http://<master node ip>:8080/```.

Under ```Github``` click "Advanced" and tick "Specify another hook URL for GitHub configuration" and insert into the box that appears:
```http://<master node ip>:8080/github-webhook/```. This will allow triggering of updates on git commits.

Click save.

## 9. Create a jenkins job

Create a jenkins job, and click ```Github Project```, enter the url of your github repo.

Under Source Code Management click ```git``` and insert the github for your repo again.

The Build Triggers will trigger a build on each github commit. Under Build Triggers tick "GitHub hook trigger for GITScm polling".

In the Build sections, create an ```execute shell``` with the following code:

```
IMAGE_NAME="<Your Dockerhub Account>/hellowhale:${BUILD_NUMBER}"
docker build . -t $IMAGE_NAME
docker login -u <Your Dockerhub Account> -p ${DOCKER_HUB}
docker push $IMAGE_NAME
```

This will build an image and push it to dockerhub on each commit. Create another ```execute shell`` and enter the below code:

```
IMAGE_NAME="<Your Dockerhub Account>/hellowhale:${BUILD_NUMBER}"
kubectl set image deployment/hellowhale hellowhale=$IMAGE_NAME
```

This will reset the deployment image on each commit. We are now finished with the pipeline, let's test it.

## 10. Testing pipeline

Make a notable change to the ```html/index.html``` in the machine. Then run:

```git add html/index.html```

Commit the changes and follow the instructions for git authentication:

```git commit -m "Change"```

Finally ```git push```. Quickly switch to the jenkins browser and click on your project. You should see a progress bar, when it is complete go to your browser and enter the LoadBalancer's ip into the search bar (as in the last step of step 6). You should see the changes you commited to Github.
