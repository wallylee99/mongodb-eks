<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="https://stackedit.io/style.css" />
</head>

<body class="stackedit">
  <div class="stackedit__html"><p>In this setup guide, I will deploy a MongoDB and Mongo Express application on AWS Kubernetes Cluster (EKS). EKS is a managed Kubernetes service that makes it easier to run Kubernetes on AWS.</p>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/eks-3.png" alt=""></p>
<p>Using EKS offers the following major benefits:</p>
<ul>
<li>Pre-install Kubernetes master nodes and other apps like container runtime for you</li>
<li>Master nodes are managed by AWS so you can focus on worker nodes</li>
<li>Service are deployed across multiple Availability Zones to ensure high availability</li>
<li>Automatic scaling by adjusting the number of work nodes in the cluster</li>
</ul>
<p>Kubernetes (K8s) is an open source platform to deploy, scale and manage containerized applications. Here my setup is to deploy a MongoDB database and Mongo Express application using their docker images from  <a href="https://hub.docker.com/">Docker Hub</a>.</p>
<h2 id="objectives">Objectives:</h2>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/mongodb-2.png" alt=""></p>
<ol>
<li>
<p>Create EKS Cluster using eksctl command
</li>
<li>
<p>Create MongoDB deployment and service using kubectl command
</li>
<li>
<p>Create Mongo Express deployment and service using kubectl command
</li>
</ol>
<h2 id="step-1-install-eksctl-command">Step 1: Install eksctl command</h2>
<p>In my previous  <a href="https://www.wallacel.com/index.php/2024/05/29/provision-an-eks-cluster-using-terraform/">blog</a>, I already explored how to create an EKS cluster using Terraform. This time I will use an open source CLI tool eksctl to create my EKS cluster. It automatically creates the necessary AWS resources (e.g. VPC, Subnets, Security Group, Node Group, etc.) required with a single command.</p>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/eksctl.png" alt=""></p>
<p>This will Install the eksctl command on MacOS.</p>
<pre class=" language-bash"><code class="prism  language-bash">brew tap weaveworks/tap
brew <span class="token function">install</span> weaveworks/tap/eksctl
</code></pre>
<p>Prepare a parameter file (<strong>eks-cluster.yaml</strong>) for creating my EKS Cluster. It contains a node group with two EC2 instances of type  <strong>t2.micro</strong>.</p>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> eksctl.io/v1alpha5
<span class="token key atrule">kind</span><span class="token punctuation">:</span> ClusterConfig

<span class="token key atrule">metadata</span><span class="token punctuation">:</span>
  <span class="token key atrule">name</span><span class="token punctuation">:</span> mongo<span class="token punctuation">-</span>cluster
  <span class="token key atrule">region</span><span class="token punctuation">:</span> ca<span class="token punctuation">-</span>central<span class="token punctuation">-</span><span class="token number">1</span>

<span class="token key atrule">nodeGroups</span><span class="token punctuation">:</span>
  <span class="token punctuation">-</span> <span class="token key atrule">name</span><span class="token punctuation">:</span> linux
    <span class="token key atrule">instanceType</span><span class="token punctuation">:</span> t2.micro
    <span class="token key atrule">desiredCapacity</span><span class="token punctuation">:</span> <span class="token number">2</span>
    <span class="token key atrule">privateNetworking</span><span class="token punctuation">:</span> <span class="token boolean important">true</span>
</code></pre>
<pre class=" language-bash"><code class="prism  language-bash">% eksctl create cluster -f eks-cluster.yaml
</code></pre>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/image-62.png" alt=""></p>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/image-64.png" alt=""></p>
<p>Use the following command to check my cluster is created.</p>
<pre class=" language-bash"><code class="prism  language-bash">% kubectl get nodes
</code></pre>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/image-66.png" alt=""></p>
<h2 id="step-2-deploy-mongodb">Step 2: Deploy MongoDB</h2>
<p>We will be using the latest MongoDB official docker  <a href="https://hub.docker.com/_/mongo">image</a>. There are two environment variables we need to setup  <code>MONGO_INITDB_ROOT_USERNAME</code>,  <code>MONGO_INITDB_ROOT_PASSWORD</code>  and I will store them in a secret file instead of hardcoding in the configuration file.</p>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/image-67.png" alt=""></p>
<p>In order to store the user name and password in a secret configuration file, first we need to encode their values from plain text to base64. For example, my user name is ‘<code>root</code>‘ and password is ‘<code>myrootpassword</code>‘, the commands are as follow.</p>
<pre class=" language-bash"><code class="prism  language-bash">% <span class="token keyword">echo</span> -n <span class="token string">'root'</span> <span class="token operator">|</span> base64

cm9vdA<span class="token operator">==</span>

% <span class="token keyword">echo</span> -n <span class="token string">'myrootpassword'</span> <span class="token operator">|</span> base64

bXlyb290cGFzc3dvcmQ<span class="token operator">=</span>
</code></pre>
<p>My  <strong>secrets.yaml</strong>  looks like this.</p>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> v1
<span class="token key atrule">kind</span><span class="token punctuation">:</span> Secret
<span class="token key atrule">metadata</span><span class="token punctuation">:</span>
    <span class="token key atrule">name</span><span class="token punctuation">:</span> mongodb<span class="token punctuation">-</span>secrets
<span class="token key atrule">type</span><span class="token punctuation">:</span> Opaque
<span class="token key atrule">data</span><span class="token punctuation">:</span>
    <span class="token key atrule">mongodb-root-username</span><span class="token punctuation">:</span> cm9vdA==
    <span class="token key atrule">mongodb-root-password</span><span class="token punctuation">:</span> bXlyb290cGFzc3dvcmQ=
</code></pre>
<p>Create the secrets in Kubernetes:</p>
<pre class=" language-bash"><code class="prism  language-bash">% kubectl apply -f secrets.yaml
% kubectl get secret
</code></pre>
<p>Now we can prepare our MongoDB deployment file (<strong>mongo.yaml</strong>) and reference the user name and password from the Kubernetes secrets.</p>
<p>Explanation:</p>
<ul>
<li>image: mongo – name of the docker image</li>
<li>containerPort: 27017 – standard port no. that Mongo DB server listens on</li>
</ul>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> apps/v1
<span class="token key atrule">kind</span><span class="token punctuation">:</span> Deployment
<span class="token key atrule">metadata</span><span class="token punctuation">:</span>
  <span class="token key atrule">name</span><span class="token punctuation">:</span> eks<span class="token punctuation">-</span>mongodb<span class="token punctuation">-</span>deployment
  <span class="token key atrule">labels</span><span class="token punctuation">:</span>
    <span class="token key atrule">app</span><span class="token punctuation">:</span> mongodb
<span class="token key atrule">spec</span><span class="token punctuation">:</span>
  <span class="token key atrule">replicas</span><span class="token punctuation">:</span> <span class="token number">1</span>
  <span class="token key atrule">selector</span><span class="token punctuation">:</span>
    <span class="token key atrule">matchLabels</span><span class="token punctuation">:</span>
      <span class="token key atrule">app</span><span class="token punctuation">:</span> mongodb
  <span class="token key atrule">template</span><span class="token punctuation">:</span>
    <span class="token key atrule">metadata</span><span class="token punctuation">:</span>
      <span class="token key atrule">labels</span><span class="token punctuation">:</span>
        <span class="token key atrule">app</span><span class="token punctuation">:</span> mongodb
    <span class="token key atrule">spec</span><span class="token punctuation">:</span>
      <span class="token key atrule">containers</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> <span class="token key atrule">name</span><span class="token punctuation">:</span> mongodb
        <span class="token key atrule">image</span><span class="token punctuation">:</span> mongo
        <span class="token key atrule">ports</span><span class="token punctuation">:</span>
        <span class="token punctuation">-</span> <span class="token key atrule">containerPort</span><span class="token punctuation">:</span> <span class="token number">27017</span>
        <span class="token key atrule">env</span><span class="token punctuation">:</span>
        <span class="token punctuation">-</span> <span class="token key atrule">name</span><span class="token punctuation">:</span> MONGO_INITDB_ROOT_USERNAME
          <span class="token key atrule">valueFrom</span><span class="token punctuation">:</span>
            <span class="token key atrule">secretKeyRef</span><span class="token punctuation">:</span>
              <span class="token key atrule">name</span><span class="token punctuation">:</span> mongodb<span class="token punctuation">-</span>secrets
              <span class="token key atrule">key</span><span class="token punctuation">:</span> mongodb<span class="token punctuation">-</span>root<span class="token punctuation">-</span>username
        <span class="token punctuation">-</span> <span class="token key atrule">name</span><span class="token punctuation">:</span> MONGO_INITDB_ROOT_PASSWORD
          <span class="token key atrule">valueFrom</span><span class="token punctuation">:</span> 
            <span class="token key atrule">secretKeyRef</span><span class="token punctuation">:</span>
              <span class="token key atrule">name</span><span class="token punctuation">:</span> mongodb<span class="token punctuation">-</span>secrets
              <span class="token key atrule">key</span><span class="token punctuation">:</span> mongodb<span class="token punctuation">-</span>root<span class="token punctuation">-</span>password
</code></pre>
<p>Now we can deploy our MongoDB pod</p>
<pre class=" language-bash"><code class="prism  language-bash">% kubectl apply -f mongo.yaml
% kubectl get pod
% kubectl describe pod <span class="token operator">&lt;</span>pod-name<span class="token operator">&gt;</span>
</code></pre>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/image-68.png" alt=""></p>
<p>Next, we will create an internal service for other pods (e.g. Mongo Express) to talk to the MongoDB server we just created.</p>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/mongodb-svc-1.png" alt=""></p>
<p>Our MongoDB service file (<strong>mongosvc.yaml</strong>) looks like this.</p>
<p>Explanation:</p>
<ul>
<li>selector: app: mongodb – specify the pod this service will attach to</li>
<li>targetPort: 27017 – match the container port no. of MongoDB deployment</li>
<li>port: 27017 – port no. this service listens on</li>
</ul>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> v1
<span class="token key atrule">kind</span><span class="token punctuation">:</span> Service
<span class="token key atrule">metadata</span><span class="token punctuation">:</span>
  <span class="token key atrule">name</span><span class="token punctuation">:</span> mongodb<span class="token punctuation">-</span>service
<span class="token key atrule">spec</span><span class="token punctuation">:</span>
  <span class="token key atrule">selector</span><span class="token punctuation">:</span>
    <span class="token key atrule">app</span><span class="token punctuation">:</span> mongodb
  <span class="token key atrule">ports</span><span class="token punctuation">:</span>
    <span class="token punctuation">-</span> <span class="token key atrule">protocol</span><span class="token punctuation">:</span> TCP
      <span class="token key atrule">port</span><span class="token punctuation">:</span> <span class="token number">27017</span>
      <span class="token key atrule">targetPort</span><span class="token punctuation">:</span> <span class="token number">27017</span>
</code></pre>
<p>Now we can create our MondoDB internal service.</p>
<pre class=" language-bash"><code class="prism  language-bash">% kubectl apply -f mongosvc.yaml
% kubectl get <span class="token function">service</span>
</code></pre>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/image-70.png" alt=""></p>
<p>To verify the service is attached to the correct MongoDB pod, we can check the IP and Port no. of our service.</p>
<pre class=" language-bash"><code class="prism  language-bash">% kubectl describe <span class="token function">service</span> mongodb-service
% kubectl get pod -o wide
</code></pre>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/image-71.png" alt=""></p>
<p>Here the  <strong>Endpoints</strong>  of our service is showing  <strong><code>192.168.83.54:27017</code></strong>, which matches the internal IP address of our MongoDB pod name  <strong><code>eks-mongodb-deployment-67f4f7cddf-2hccf</code></strong></p>
<h2 id="step-2-deploy-mongo-express">Step 2: Deploy Mongo Express</h2>
<p>Mongo Express is a web-based MongoDB admin interface written in Node.js, Express.js and Bootstrap 3. Now we will prepare a deployment file to deploy this app to our EKS cluster. Again, we are using the official docker image from  <a href="https://hub.docker.com/_/mongo-express">dockerhub</a>.</p>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/image-72.png" alt=""></p>
<p>There are three environment variables that we need to configure for this app:</p>
<ul>
<li><code>ME_CONFIG_MONGODB_ADMINUSERNAME</code></li>
<li><code>ME_CONFIG_MONGODB_ADMINPASSWORD</code></li>
<li><code>ME_CONFIG_MONGODB_SERVER</code></li>
</ul>
<p>We will use the same username and password defined in our MongoDB setup and we have already stored them as Kubernetes secrets. We also need to tell Mongo Express the database server to connect to by specifying the internal service name (i.e.  <strong>mongodb-service</strong>) in the  <code>ME_CONFIG_MONGODB_SERVER</code>  variable.</p>
<p>We will create a Kubernetes ConfigMap (<strong>configmap.yaml</strong>) to store this environment configuration.</p>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> v1
<span class="token key atrule">kind</span><span class="token punctuation">:</span> ConfigMap
<span class="token key atrule">metadata</span><span class="token punctuation">:</span>
  <span class="token key atrule">name</span><span class="token punctuation">:</span> mongodb<span class="token punctuation">-</span>configmap
<span class="token key atrule">data</span><span class="token punctuation">:</span>
  <span class="token key atrule">database_server</span><span class="token punctuation">:</span> mongodb<span class="token punctuation">-</span>service
</code></pre>
<p>Create the config map in Kubernetes.</p>
<pre class=" language-bash"><code class="prism  language-bash">% kubectl apply -f configmap.yaml
% kubectl get configmap
</code></pre>
<p>Now we can prepare the deployment file (<strong>express.yaml</strong>) for Mongo Express.</p>
<p>Explanation:</p>
<ul>
<li>containerPort: 8081 – standard port no. that Mongo Express listens on</li>
<li>configMapKeyRef: name: mongodb-configmap – reference the environment variable from ConfigMap</li>
</ul>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> apps/v1
<span class="token key atrule">kind</span><span class="token punctuation">:</span> Deployment
<span class="token key atrule">metadata</span><span class="token punctuation">:</span>
  <span class="token key atrule">name</span><span class="token punctuation">:</span> eks<span class="token punctuation">-</span>mongo<span class="token punctuation">-</span>express<span class="token punctuation">-</span>deployment
  <span class="token key atrule">labels</span><span class="token punctuation">:</span>
    <span class="token key atrule">app</span><span class="token punctuation">:</span> mongo<span class="token punctuation">-</span>express
<span class="token key atrule">spec</span><span class="token punctuation">:</span>
  <span class="token key atrule">replicas</span><span class="token punctuation">:</span> <span class="token number">1</span>
  <span class="token key atrule">selector</span><span class="token punctuation">:</span>
    <span class="token key atrule">matchLabels</span><span class="token punctuation">:</span>
      <span class="token key atrule">app</span><span class="token punctuation">:</span> mongo<span class="token punctuation">-</span>express
  <span class="token key atrule">template</span><span class="token punctuation">:</span>
    <span class="token key atrule">metadata</span><span class="token punctuation">:</span>
      <span class="token key atrule">labels</span><span class="token punctuation">:</span>
        <span class="token key atrule">app</span><span class="token punctuation">:</span> mongo<span class="token punctuation">-</span>express
    <span class="token key atrule">spec</span><span class="token punctuation">:</span>
      <span class="token key atrule">containers</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> <span class="token key atrule">name</span><span class="token punctuation">:</span> mongo<span class="token punctuation">-</span>express
        <span class="token key atrule">image</span><span class="token punctuation">:</span> mongo<span class="token punctuation">-</span>express
        <span class="token key atrule">ports</span><span class="token punctuation">:</span>
        <span class="token punctuation">-</span> <span class="token key atrule">containerPort</span><span class="token punctuation">:</span> <span class="token number">8081</span>
        <span class="token key atrule">env</span><span class="token punctuation">:</span>
        <span class="token punctuation">-</span> <span class="token key atrule">name</span><span class="token punctuation">:</span> ME_CONFIG_MONGODB_ADMINUSERNAME
          <span class="token key atrule">valueFrom</span><span class="token punctuation">:</span>
            <span class="token key atrule">secretKeyRef</span><span class="token punctuation">:</span>
              <span class="token key atrule">name</span><span class="token punctuation">:</span> mongodb<span class="token punctuation">-</span>secrets
              <span class="token key atrule">key</span><span class="token punctuation">:</span> mongodb<span class="token punctuation">-</span>root<span class="token punctuation">-</span>username
        <span class="token punctuation">-</span> <span class="token key atrule">name</span><span class="token punctuation">:</span> ME_CONFIG_MONGODB_ADMINPASSWORD
          <span class="token key atrule">valueFrom</span><span class="token punctuation">:</span> 
            <span class="token key atrule">secretKeyRef</span><span class="token punctuation">:</span>
              <span class="token key atrule">name</span><span class="token punctuation">:</span> mongodb<span class="token punctuation">-</span>secrets
              <span class="token key atrule">key</span><span class="token punctuation">:</span> mongodb<span class="token punctuation">-</span>root<span class="token punctuation">-</span>password
        <span class="token punctuation">-</span> <span class="token key atrule">name</span><span class="token punctuation">:</span> ME_CONFIG_MONGODB_SERVER
          <span class="token key atrule">valueFrom</span><span class="token punctuation">:</span> 
            <span class="token key atrule">configMapKeyRef</span><span class="token punctuation">:</span>
              <span class="token key atrule">name</span><span class="token punctuation">:</span> mongodb<span class="token punctuation">-</span>configmap
              <span class="token key atrule">key</span><span class="token punctuation">:</span> database_server
</code></pre>
<p>Deploy our Mongo Express pod</p>
<pre class=" language-bash"><code class="prism  language-bash">% kubectl apply -f express.yaml
% kubectl get pod
% kubectl describe pod <span class="token operator">&lt;</span>mongo_express_pod_name<span class="token operator">&gt;</span>
</code></pre>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/image-73.png" alt=""></p>
<p>Our final step is to allow access to Mongo Express through browser by creating an external service.</p>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/express-svc-1.png" alt=""></p>
<p>Our Mongo Express service configuration file (<strong>expresssvc.yaml</strong>) looks like this.</p>
<p>Explanation:</p>
<ul>
<li>type: LoadBalance – allows to accept external request by assigning an external IP address</li>
<li>nodePort: 30000 – port no. for the external IP address to listen on</li>
<li>targetPort: 8081 – match the container port no. of Mongo Express deployment</li>
<li>port: 8081 – port no. this service listens on</li>
</ul>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> v1
<span class="token key atrule">kind</span><span class="token punctuation">:</span> Service
<span class="token key atrule">metadata</span><span class="token punctuation">:</span>
  <span class="token key atrule">name</span><span class="token punctuation">:</span> mongo<span class="token punctuation">-</span>express<span class="token punctuation">-</span>service
<span class="token key atrule">spec</span><span class="token punctuation">:</span>
  <span class="token key atrule">selector</span><span class="token punctuation">:</span>
    <span class="token key atrule">app</span><span class="token punctuation">:</span> mongo<span class="token punctuation">-</span>express
  <span class="token key atrule">type</span><span class="token punctuation">:</span> LoadBalancer  
  <span class="token key atrule">ports</span><span class="token punctuation">:</span>
    <span class="token punctuation">-</span> <span class="token key atrule">protocol</span><span class="token punctuation">:</span> TCP
      <span class="token key atrule">port</span><span class="token punctuation">:</span> <span class="token number">8081</span>
      <span class="token key atrule">targetPort</span><span class="token punctuation">:</span> <span class="token number">8081</span>
      <span class="token key atrule">nodePort</span><span class="token punctuation">:</span> <span class="token number">30000</span>
</code></pre>
<p>Deploy our external service for Mongo Express.</p>
<pre class=" language-bash"><code class="prism  language-bash">% kubectl apply -f expresssvc.yaml
% kubectl get <span class="token function">service</span>
</code></pre>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/image-75.png" alt=""></p>
<p>An external IP is automatically assigned to the mongo-express-service just created. We can now access the Mongo Express UI from our browser (first login are “<code>admin</code>” for username and “<code>pass</code>” for password).</p>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/image-76.png" alt=""></p>
<p><img src="https://www.wallacel.com/wp-content/uploads/2024/06/image-77.png" alt=""></p>
<h2 id="clean-up">Clean Up</h2>
<p>Resources created in this guide will incur charges if you keep them running. We can easily delete the EKS cluster and all its resources (e.g. EC2 instances, VPC, subnets, node group, etc.) with the eksctl command.
<pre><code>% eksctl delete cluster --name &lt;cluster name&gt;
</code></pre>
</div>
</body>

</html>
