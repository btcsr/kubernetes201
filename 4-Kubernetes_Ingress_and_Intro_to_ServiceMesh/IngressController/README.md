### Carry out following deployments first.

#### Create the Namespace.
```
$ curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml \
    | kubectl apply -f -

```

#### Create default Backend for the Nginx ingress controller.
```
$ curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml \
    | kubectl apply -f -

```

#### Create ConfigMap for the Nginx ingress controller.
```
$ curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml \
    | kubectl apply -f -
    
$ curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml \
    | kubectl apply -f -

$ curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml \
    | kubectl apply -f -
```

#### Set the RBAC rules.
```
$ curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml \
    | kubectl apply -f -
```

Create the `Nginx ingress controller` configuration file as shown below.
```
$ vi ingress-controller.yaml

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx 
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ingress-nginx
  template:
    metadata:
      labels:
        app: ingress-nginx
      annotations:
        prometheus.io/port: '10254'
        prometheus.io/scrape: 'true'
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      hostNetwork: true
      containers:
        - name: nginx-ingress-controller
          image: gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.15
          args:
            - /nginx-ingress-controller
            - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
          - name: http
            containerPort: 80
          - name: https
            containerPort: 443

```

Deploy the Nginx Ingress controller.
```
$ kubectl create -f ingress-controller.yaml
```
### Blue and Green application

Create and deploy the Blue application from following configuration file.
```
$ vim blue.yaml


---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: blue
spec:
  template:
    metadata:
      labels:
        app: blue
    spec:
      containers:
      - name: blue
        image: teamcloudyuga/blue
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: blue
  labels:
    app: blue
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: blue


```
Deploy the application
```
$ kubectl create -f blue.yaml
```
Create and deploy the Green application from following configuration file.

```
$ vim green.yaml


---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: green
spec:
  template:
    metadata:
      labels:
        app: green
    spec:
      containers:
      - name: green
        image: teamcloudyuga/green
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: green
  labels:
    app: green
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: green

```

Deploy the Green application.
```
$ kubectl create -f green.yaml
```

Create a Path based ingress object.
```
$ vim ingress_path.yaml



apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: path
  annotations:
    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: cy.myweb.com
    http:
      paths:
      - path: /blue
        backend:
          serviceName: blue
          servicePort: 80
      - path: /green
        backend:
          serviceName: green
          servicePort: 80

```

Deploy this ingress object.
```
$ kubectl create -f ingress_path.yaml
```

Get the status of ingress.
```
$ kubectl get ing
NAME      HOSTS          ADDRESS           PORTS     AGE
path      cy.myweb.com   165.227.120.162   80        25m
```

 Edit the `/etc/hosts` file and create records of `cy.myweb.com` with above shown address for me it is `165.227.120.162`.
 
 Curl to the `cy.myweb.com/blue` and see the output of curl.
```
$ curl cy.myweb.com/blue
<!DOCTYPE html>
<html>
<body bgcolor="Blue">
<h1> This is Blue Application <h1>
<h1>Hello From Cloudyuga!</h1>
<p><a href="https://cloudyuga.guru/"> Visit cloudyuga.guru!</a></p>



</body>
</html>

```
You can aslo check the in the browser `cy.myweb.com/web` will show you nginx running.

Curl to the `cy.myweb.com/green` and see the output of curl.
```
$ curl cy.myweb.com/green
<!DOCTYPE html>
<html>
<body bgcolor="Green">
<h1> This is Green Application <h1>
<h1>Hello From Cloudyuga!</h1>
<p><a href="https://cloudyuga.guru/"> Visit cloudyuga.guru!</a></p>



</body>
</html>

```
You can also see the application in browser by using hostname `cy.myweb.com/green` and `cy.myweb.com/blue`


## Delete Services, Deployments and Ingress.
```
$ kubectl delete deploy blue green
$ kubectl delete svc blue green
$ kubectl delete ing path
```



