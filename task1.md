## Task 1





Please describe in as much detail as you can the process of *one* out of the two scenarios:

A user has written a YAML file containing a Kubernetes `Deployment` resource containing the following resources:
- a Kubernetes `Service` resource making the deployment to get exposed
- a kubernetes `ingress` resource linking to the deployment
- a kubernetes `secret` resource referenced in deployment.

What happens underneath on the local machine, within the cluster and on the network?


---------------------------------------------------------------------------------------------------------------------


mkdir ~/.kube

kubectl version

kubectl create namespace jenkinsprod

A deployment is responsible for keeping a set of pods running


deployment.yaml


```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: jenkins-deploymment
  template:
    metadata:
      labels:
        app: jenkins-deployment
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
	imagePullPolicy: "Always"
        ports:
          - name: http-port-ui
            containerPort: 8080
          - name: jnlp-port-hooks
            containerPort: 50000
      imagePullSecrets:
      - name: registrykey
	env:
       - name: JENKINS_ROOT_PASSWORD
         valueFrom:
           secretKeyRef:
             name: jenkins-root-password
             key: ryd$@!!!!ryd
        envFrom:
        - secretRef:
            name: jenkins-user-credentials
        volumeMounts:
          - name: jenkinsconfigdata
            mountPath: /var/jenkinsdata
	  - name: jenkinsdata
	    mountPath: /data/jenkinsdata
	  - name: docker-socket
	   mountPath: /var/run/docker.sock
      volumes:
        - name: jenkinsconfigdata
          emptyDir: {}
	- name: jenkinsdata
	env:
        - name: MESSAGE
          value: Deployed successfully

```




kubectl create -f jenkins.yaml --namespace jenkinsprod

kubectl get pods -n jenkinsprod




A service is responsible for enabling network access to a set of pods



 create ingress rules that expose your deployment to the external world
 
 
 
 
 
 kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
 
 kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud-generic.yaml

Kubernetes ingress is an "object that manages external access to services in a cluster, typically through HTTP". With an ingress, you can support load balancing, TLS termination, and name-based virtual hosting from within your cluster

Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the Ingress resource.


If you want session affinity on pod-to-service routing, you can set the SessionAffinity: ClusterIP field on a Service object



service1.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  labels:
    name: jenkins-service
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
  selector:
    app: jenkins-deployment
  type: Nodeport

```

Note : selector type can be ClusterIp or Nodeport

then access via http://jenkinsurl:30080/jenkins-service



Alternative way for service1.yaml


```
kind: Service
apiVersion: v1
metadata:
  name: jenkins-service
spec:
  selector:
    app: jenkins-deployment
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30080
  clusterIP: 10.0.171.200
  loadBalancerIP: 192.162.24.1
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
      - ip: 146.143.45.160


```














service2.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service-jnlp
  labels:
    name: jenkins-service-jnlp
spec:
  type: ClusterIP
  ports:
    - port: 50000
      targetPort: 50000
      nodeport: 50080   
  selector:
    app: jenkins-deployment
  type: Nodeport

```


http://jenkinsurl:50080/jenkins-service-jnlp

Note : selector type can be  Nodeport othe than ClusterIp

then in spec remove type just mention port,targetport,nodeport only mention slector type as Nodeport


kubectl create -f service1.yaml --namespace jenkinsprod  or kubectl apply -f service1.yaml

kubectl create -f service2.yaml --namespace jenkinsprod  or kubectl apply -f service2.yaml

kubectl get services --namespace jenkinsprod

kubectl get service 

kubectl get service jenkins

kubectl get nodes -o wide

open a web browser and navigate to http://your_external_ip:30000


kubectl get pods -n ingress-nginx

kubectl get svc



To install the Nginx Ingress Controller

helm install nginx-ingress stable/nginx-ingress --set controller.publishService.enabled=true

ingress.yaml

```

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins-ingress
spec:
  rules:
  - http:
      paths:
      - path: /data
        backend:
          serviceName: jenkins-service
          servicePort: 8080
      - path: /data
        backend:
          serviceName: jenkins-service-jnlp
          servicePort: 50000
	  
```


kubectl apply -f ingress.yaml

kubectl get svc -n ingress-nginx ingress-nginx -o=jsonpath='{.status.loadBalancer.ingress[0].ip}' or kubectl get services -o wide -w nginx-ingress-controller


Note : 

Under rules we can add host: domain name or the url


jenkins-secrets.yaml

```
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-root-password
type: Opaque
data:
  password: ryd$@!!!!ryd
  
```


kubectl apply -f jenkins-secrets.yaml

kubectl get secret jenkins-root-password -o jsonpath='{.data.password}'

kubectl get secret mariadb-root-password -o jsonpath='{.data.password}' | base64 --decode




or 

kubectl create secret generic registrykeygeneric --from-literal=username=jenkins --from-literal=password=ryd$@!!!!ryd


we can add this in deployment.yaml

```
        env:
        - name: REGISTRY_USERNAME
          valueFrom:
            secretKeyRef:
              name: registrykeygeneric
              key: username
        - name: REGISTRY_PASSWORD
          valueFrom:
            secretKeyRef:
              name: registrykeygeneric
              key: password



```







if setup is done on AWS or Azure


add this

```
      volumes:
        persistentVolumeClaim:
        claimName: jenkinsdata
	Persistence:
          StorageClass: efs
```



create & apply pvc.yaml



```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkinsdata
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp2 
  resources:
    requests:
      storage: 50Gi


```





### Alternative usecase on prometheus


deployment.yaml


```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-server
spec:
  replicas: 1
  template:
    spec:
      serviceAccountName: prometheus-server
      containers:
        - name: prometheus
          image: prom/prometheus
          args:
            - "--config.file=/etc/prometheus/prometheus.yaml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/
      volumes:
        - name: prometheus-config-volume
          configMap:
            name: prometheus-configuration
        - name: prometheus-storage-volume
          emptyDir: {}



```




service.yaml

```
kind: Service
apiVersion: v1
metadata:
  labels:
    app: prometheus
  name: prometheus
spec:
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30100
  selector:
    app: prometheus
  type: NodePort


```



http://prometheusurl:30100




## What happens underneath on the local machine, within the cluster and on the network?




Kubernetes manages networking through CNI’s on top of docker 

Container Networking Interface - defined interface that can be called by kubernetes to execute actions to provide networking functionality.

CNI - Flannel I choose

k8s does not use docker0(172.0.0.0)


Docker ---- uses Overlay networks such as vxlan or ipsec



Each Service has a setting called ServiceType that defines how that service is exposed. 
You can set this to ClusterIP, NodePort, LoadBalancer, or ExternalName depending on your particular deployment scenario


ClusterIP :

ClusterIP is the default ServiceType and it creates a single IP address that can be used to access its Pods which can only be accessed from inside the cluster. If KubeDNS is enabled it will also get a series of DNS records assigned to it include an A record to match its IP. This is very useful for exposing microservices running inside the same Kubernetes cluster to each other.


Want applications in the same Kubernetes cluster to talk to each other ------> then you would use ClusterIP as there is no need to expose it to the outside world



Nodeport:

NodePort builds on top of ClusterIP to create a mapping from each Worker Node’s static IP on a specified (or Kubernetes chosen) Port. A Service exposed as a NodePort can be accessed via <node-ip-address>:<node-port>. This ServiceType can be useful when developing applications with minikube or for exposing a specific Port to an application via an unmanaged load balancer or round robin DNS.
	
	
Do not have a supported Load Balancer  ------> then  should look at NodePort
	
	
LoadBalancer:

LoadBalancer builds on top of NodePort and is used to automatically configure a supported external Load Balancer (for instance an ELB in Amazon) to route traffic through to the NodePort of the Service. This is the most versatile of the ServiceTypes but requires that you have a supported Load Balancer in your infrastructure of which most major cloud providers have.




Node01 -----> eth0 -----> cbr0 -----> veth0 -----> pod -----> container

Each pod’s network namespace communicates with the node’s root netns through a virtual ethernet pipe. On the node side, this pipe appears as a device that typically begins with veth and ends in a unique identifier.

kube-proxy can configure IPVS to handle the translation of virtual Service IPs to pod IPs


## References 

https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/
