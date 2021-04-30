Microfrotend deployments (Multiple Angular Frontends from a single portal)

![](/images/2021-04-29-Securing_MicroFrontend_Deployment_in_Kuberentes/media/image1.png)

In continuation of my Part-1 learn guide [Full Stack Deployment in Kubernetes (Angular, IdentityServer4, Dotnet Core, MongoDB, CIVO)](https://www.civo.com/learn/user_guides/446) , I would like to share my experience, how I took my idea to the next level using CIVO and the tools I have used for the same. To be more precise, before I continue with by Too Long to Kindly Read statements, I am going to talk about below points:

1.  Configuring a secure domain url

2.  Using tools like Rancher as Service & POPEYE

3.  An error I face while I setup micro-frontend in Kubernetes

**Too Long Kindly Read**

I always had this weird instinct that I have the potential to build something and it will revolutionize the world. Being in software industry I tried to look out for the opportunity but besides being busy I couldn’t much invest into the “building” part. That was when I came across CIVO’s k3s cluster beta program. Instantly I started investing time to learn k8s and tried out different tools in CIVO environment. As part of that I begin to setup and launch a full stack app into CIVO cluster. I was so excited that it even inspired me to take a couple of courses and I tried to extend and deploy the final projects into a production ready application and deploy them on the k3s cluster in CIVO.

1.  I purchased the domain “https://happilypresents.co.in/” and configured the CIVO DNS using the [learn guide](https://www.civo.com/learn/configure-dns).

The application is a set of microservices, one angular frontend that connects to three backend services for identity, book and chapter services using bearer tokens. I used “helm” to deploy the frontend and identity services, “openfaas” deploy for a couple of microservices. After the first part, I tried to secure the application using Let’s Encrypt using the learning guide [here](https://www.civo.com/learn/get-a-wildcard-certificate-with-cert-manager-and-civo-dns). I haven’t used the wild card issuer; I have used the conventional steps as below:

1.  Create the issuer

2.  Deploy the ingress with the secret key

3.  Troubleshoot whether the certificate is the issued from Let’s Encrypt (stage/prod api)

kubectl apply -f ./cert-manager/civoissuer.prod.yaml

kubectl apply -f ingress-cert-civo.yaml

kubectl get issuer -n \<your namespace\>

kubectl get certificates kitchen-n \<your namespace\>

kubectl describe certificates \<your secret\> -n \<your namespace\>

  - At this step I got an error “certificate public key must be different than account key”.

  - I removed the certificates and repeated the steps with unique secret key which is different from the issuer.yaml

<!-- end list -->

2.  For Rancher as Service, I have done a video [here](https://www.youtube.com/watch?v=GStqMCW0B5g), its basically to taking care of my deployed services and cluster as a whole. The free life time access is available at https://kubernautic.com/

I have used [POPEYE](https://popeyecli.io/) to sanitize the cluster, the command and usage:

> wget https://github.com/derailed/popeye/releases/download/v0.9.0/popeye\_Linux\_x86\_64.tar.gz
> 
> tar -xvf popeye\_Linux\_x86\_64.tar.gz
> 
> mv popeye /usr/local/bin/
> 
> POPEYE\_REPORT\_DIR=/mnt/e/Kubernetes/ popeye --save --out html --output-file report.html

3.  After setting up everything my next level was to gain knowledge on a few frontend framework/library like Angular/React and build a micro- front-end based architecture, I am using the [single-spa](https://single-spa.js.org/) framework to bring together a couple of websites into CIVO. So far I am able to build and deploy the package, only thing is getting stuck with the ingress setup,

![](/images/2021-04-29-Securing_MicroFrontend_Deployment_in_Kuberentes/media/image2.png)

Later, I fixed the issue by implementing the ImportMap deployer.

During this application development I had issues with my local environment and my [Part-1 learning guide](https://www.civo.com/learn/user_guides/446) helped me to setup everything from scratch which had all the bookmarks and commands. Hope this learning guide will help us motivate to spend tiny bit of time to build the application in a state of the art fashion and take it to the next level. I am planning to get “terraform” to spin up the cluster and deploy the application instantly without any manual intervention.

**Conclusion**

Overall, my experience with CIVO to build microservice and micro-frontends led to develop a secure happilypresents.co.in application. My future plans are to enhance it with the industry standard best practices available in CIVO marketplace and contribute to the community positively.

You can always visit my application at <https://happilypresents.co.in/> and the code repo is @ [Kitchen](https://github.com/rajeshradhakrishnanmvk/kitchen.git) – “frontend” branch

![](/images/2021-04-29-Securing_MicroFrontend_Deployment_in_Kuberentes/media/image3.png)

Figure : Micro-Frontend pods

![](/images/2021-04-29-Securing_MicroFrontend_Deployment_in_Kuberentes/media/image4.png)

Figure : Micro-Frontend services

![](/images/2021-04-29-Securing_MicroFrontend_Deployment_in_Kuberentes/media/image5.png)

Figure : Micro-Frontend Ingress

![](/images/2021-04-29-Securing_MicroFrontend_Deployment_in_Kuberentes/media/image6.png)

Figure : OpenFaas pods

![](/images/2021-04-29-Securing_MicroFrontend_Deployment_in_Kuberentes/media/image7.png)

Figure : OpenFaas services

![](/images/2021-04-29-Securing_MicroFrontend_Deployment_in_Kuberentes/media/image8.png)

Figure : HappilyPresents.co.in Ingress
