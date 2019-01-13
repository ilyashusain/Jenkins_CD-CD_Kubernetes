# Jenkins ci-cd k8s Pipeline

In this guide we will create a Jenkins K8s pipeline with full ci-cd integration for a sample webpage app.

## 1. Create a node

Create a node and configure the firewalls to pass in http traffic and jenkins on port ```8080```.

## 2. Install prerequisite plugins

Install java, wget, git, docker and kubernetes with the following command:

```sudo yum install -y java wget git docker kubectl```

## 3. Configure docker

Configure docker by placing root user into the docker group. This way, we will not have to use ```sudo``` every time we try to run a docker command. First, create the docker group:

```sudo groupadd docker```

Next, place root user into the group:

```sudo usermod -aG docker $USER```

Now restart the node through GCP so new user permissions take effect.

## 4. Install jenkins

Install jenkins (CentOS 7):

```sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo```

```sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key```

```sudo yum install -y jenkins```

Next, place the jenkins user into the docker group. That way jenkins can run docker commands without sudo (important since jenkins cannot run sudo commands in the first place):

```sudo usermod -aG docker jenkins```

Restart the vm again so new user permissions take effect. Now go into the vm and start the jenkins service:

```sudo service jenkins start```

Then copy the vms ip into the browser as ```<vms ip>:8080>``` and follow the instructions to setting up a user account for jenkins.

## 5. Setting up docker

Start the docker service:

```sudo service docker start```

Clone the above repository with git clone:

```git clone https://github.com/ilyashusain/Jenkins_CD-CD_Kubernetes.git```

Then ```cd``` into this repository with:

```cd Jenkins_CD-CD_Kubernetes/```

and build the docker image and push to dockerhub:

```docker build . -t hellowhale```

```docker tag hellowhale ilyashusain/hellowhale```

```docker login -u ilyashusain -p <Your Dockerhub Password>```

```docker push ilyashusain/hellowhale```

## 6. Create k8s cluster for hellowhale deployment

Authenticate with google cloud and follow the instructions:

```gcloud auth login```

Create a cluster of size 1 with:

```gcloud container clusters create pipeline5 --zone europe-west2-c --num-nodes 1```

Create a deployment for the docker image:

```kubectl create deployment hellowhale --image ilyashusain/hellowhale```

Expose the deployment with ```type LoadBalancer```:

```kubectl expose deployment/hellowhale --port 80 --name hellowhalesvc --type LoadBalancer```

Run ```kubectl get svc``` to see the ip address for the above LoadBalancer, enter it into the browser to see the webpage.

## 7. Configure jenkins in the console

Copy the ```/home/$USER/.kube/config``` file to ```/var/lib/jenkins/.kube```. If the ```/.kube``` directory in ```/var/lib/jenkins/.kube``` does not exist then create it. Use sudo when copying, as such:

```sudo cp /home/$USER/.kube/config /var/lib/jenkins/.kube/```

Change ownership of the above file so that jenkins owns it and make it executable, without doing so we cannot run ```kubectl``` commands:

```sudo chown jenkins:jenkins /var/lib/jenkins/.kube/config```

```chmod +x /var/lib/jenkins/.kube/config ```

Now restart the jenkins service:

```sudo service jenkins restart```

## 8. Configure jenkins in the browser

Once you have logged in to jenkins, navigate to ```Manage Jenkins > Configure System```. Under ```Global Properties``` tick environment variables and put ```DOCKER_HUB``` under ```Name``` and your dockerhub password under ```Value```.

Under ```Jenkins Location``` in the ```Jenkins URL``` put ```http://<vm ip where jenkins installed>:8080/```.

Under ```Github``` click "Advanced" and tick "Specify another hook URL for GitHub configuration" and insert into the box that appears:
```http://<vm ip where jenkins installed>:8080/github-webhook/```. This will allow triggering of updates on git commits.

Click save.

## 9. Create a jenkins job

Create a jenkins job, and click ```Github Project```, enter the url of this your github repo.

Under Source Code Management click ```git``` and insert the github for this repo again.

The Build Triggers will trigger a build on each github commit. Under Build Triggers tick "GitHub hook trigger for GITScm polling".

In the Build sections, create an ```execute shell``` with the following code:

```IMAGE_NAME="ilyashusain/hellowhale:${BUILD_NUMBER}"```

```docker build . -t $IMAGE_NAME```

```docker login -u ilyashusain -p ${DOCKER_HUB}```

```docker push $IMAGE_NAME```

This will build an image and push it to dockerhub on each commit.

```IMAGE_NAME="ilyashusain/hellowhale:${BUILD_NUMBER}"```

```kubectl set image deployment/hellowhale hellowhale=$IMAGE_NAME```

This will reset the deployment image on each commit. We are now finished with the pipeline, let's test it.

## 10. Testing pipeline

Make a change to the ```html/index.html```. Then run:

```git add html/index.html```

Commit the changes and follow the instructions for git authentication:

git commit -m "Change"

Finally ```git push```. Switch to the jenkins browser and you should see a progress bar, when it is complete go to your browser and enter the pods ip into the browser as in the last step of step 6. You should see your changes.

