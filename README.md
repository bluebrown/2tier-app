# Kubernetes - Bootstrap Declarative Projects Imperative

There are several way resources can be created in Kubernetes. It is best practice to do it declarative via apply and yaml files. However, it can be a tedious task to write dozens of lines of error prone yaml by hand.

In this post I am going to explore options to to utilize kubectl imperative commands to bootstrap the declarative yaml files.  You can find the full code [on github](https://github.com/bluebrown/2tier-app).

## Dry Run

They key to this method is the `dry-run` flag. It allows to run the given command only partially. When paired with the `-o` flag we can generate a resource definition without actually creating the resource.

```shell
kubectl run my-app --image nginx --dry-run=client -o yaml
```

{% details Output %}

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: my-app
  name: my-app
spec:
  containers:
  - image: nginx
    name: my-app
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

{% enddetails %}

## Set

We can take this a step further and set more fields that cant be set on the normal run or create command.

```shell
kubectl run my-app --image nginx --dry-run=client -o yaml \
  | kubectl set resources --limits="cpu=100m,memory=265Mi" --local -f - -o yaml
```

{% details Output %}

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: my-app
  name: my-app
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: my-app
    resources:
      limits:
        cpu: 100m
        memory: 265Mi
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  terminationGracePeriodSeconds: 30
status: {}
```

{% enddetails %}

All whats left is writing the output to a file and modifying it before running it declarative vial `apply`.

## Bootstrapping a 2 Tier App

Lets bootstrap a 2 tier application. The first tier will be nginx serving static content from a configMap and the second tier will be a nodejs backend running json-server.

```shell
mkdir objects
```

### Backend

First generate the backend deployment definition by creating a deployment imperatively in dry-run mode, setting its resource limits and finally writing to a yaml file.

```shell
kubectl create deploy backend --image bluebrown/json-server --port 3000 --dry-run=client -o yaml \
    | k set resources --limits="cpu=100m,memory=256Mi" --local  -f - -o yaml \
    | tee objects/backend.deploy.yaml
```

{% details Output %}

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: backend
  name: backend
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: backend
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: backend
    spec:
      containers:
      - image: bluebrown/json-server
        imagePullPolicy: Always
        name: json-server
        ports:
        - containerPort: 3000
          protocol: TCP
        resources:
          limits:
            cpu: 100m
            memory: 256Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status: {}
```

{% enddetails %}

### Frontend

For the frontend we are following the same procedure but we need to edit the generated yaml file to declare the volume mount of the configmap.

### ConfigMap

The configMap is generated from an html file and mount it into an nginx container.

```shell
mkdir assets
vim assets/index.html
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Hello, Kubernetes</title>
  <script type="module" defer>
    const app = document.getElementById('app');
    (async () => {
      const posts = await (await fetch('http://backend/posts')).json()
      for (const { id, title, author } of posts) {
        app.append(Object.assign(
          document.createElement('p'),
          { textContent: title },
        ))
      }
    })().catch(console.warn)
  </script>
</head>
<body>
  <div id="app"></div>
</body>
</html>
```

Create a configmap from this file.

Now point point to the file to crate the config map. The filename will be used as data key.

```shell
kubectl create configmap frontend-data --from-file assets/index.html --dry-run=client -o yaml \
  | tee objects/frontend.configmap.yaml
```

{% details Output %}

```yaml
apiVersion: v1
data:
  index.html: |-
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <meta http-equiv="X-UA-Compatible" content="IE=edge">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>Hello, Kubernetes</title>
      <script type="module" defer>
        const app = document.getElementById('app');
        (async () => {
          const posts = await (await fetch('http://backend/posts')).json()
          for (const { id, title, author } of posts) {
            app.append(Object.assign(
              document.createElement('p'),
              { textContent: title },
            ))
          }
        })().catch(console.warn)
      </script>
    </head>
    <body>
      <div id="app"></div>
    </body>
    </html>
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: frontend-data
```

{% enddetails %}

### Deployment

Next, create a deployment using nginx as image.

```shell
kubectl create deploy frontend --image nginx --port 80 --dry-run=client -o yaml \
    | kubectl set resources --limits="cpu=100m,memory=256Mi" --local  -f - -o yaml \
    | tee objects/frontend.deploy.yaml
```

Now we can modify the generated frontend.deploy.yaml and add the volume section to it.

```shell
vim frontend.deploy.yaml
```

```yaml
...
containers:
- image: nginx
  ...
  volumeMounts:
  - name: frontend-data
    mountPath: /usr/share/nginx/html/
volumes:
  - name: frontend-data
    configMap:
      name: frontend-data
...
```

{% details Complete File %}

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: frontend
  name: frontend
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: frontend
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: frontend
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources:
          limits:
            cpu: 100m
            memory: 256Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - name: frontend-data
          mountPath: /usr/share/nginx/html/
        volumes:
        - name: frontend-data
          configMap:
            name: frontend-data
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status: {}
```

{% enddetails %}

### Expose the pods

Lastly we create service definitions for the deployments. We provide 2 additional flags for the backend service in order ro remap port 3000 from the container to port 80.

```shell
kubectl expose -f objects/backend.deploy.yaml --port 80 --target-port 3000 --dry-run=client -o yaml \
  | tee objects/backend.svc.yaml
```

{% details Output %}

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: backend
  name: backend
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app: backend
status:
  loadBalancer: {}
```

{% enddetails %}

```shell
kubectl expose -f objects/frontend.deploy.yaml --dry-run=client -o yaml \
  | tee objects/frontend.svc.yaml
```

{% details Output %}

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: frontend
  name: frontend
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: frontend
status:
  loadBalancer: {}
```

{% enddetails %}

### Apply

Lets apply all files at once declarative. We can even use dry-run on this command to check if the files are all ok.

```shell
kubectl apply -f objects/ --dry-run=client -o yaml \
  | tee assets/dry-run.yaml
```

{% details Output %}

```yaml
apiVersion: v1
items:
- apiVersion: apps/v1
  kind: Deployment
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"creationTimestamp":null,"labels":{"app":"backend"},"name":"backend","namespace":"default"},"spec":{"progressDeadlineSeconds":600,"replicas":1,"revisionHistoryLimit":10,"selector":{"matchLabels":{"app":"backend"}},"strategy":{"rollingUpdate":{"maxSurge":"25%","maxUnavailable":"25%"},"type":"RollingUpdate"},"template":{"metadata":{"creationTimestamp":null,"labels":{"app":"backend"}},"spec":{"containers":[{"image":"bluebrown/json-server","imagePullPolicy":"Always","name":"json-server","ports":[{"containerPort":3000,"protocol":"TCP"}],"resources":{"limits":{"cpu":"100m","memory":"256Mi"}},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File"}],"dnsPolicy":"ClusterFirst","restartPolicy":"Always","schedulerName":"default-scheduler","securityContext":{},"terminationGracePeriodSeconds":30}}},"status":{}}
    creationTimestamp: null
    labels:
      app: backend
    name: backend
    namespace: default
  spec:
    progressDeadlineSeconds: 600
    replicas: 1
    revisionHistoryLimit: 10
    selector:
      matchLabels:
        app: backend
    strategy:
      rollingUpdate:
        maxSurge: 25%
        maxUnavailable: 25%
      type: RollingUpdate
    template:
      metadata:
        creationTimestamp: null
        labels:
          app: backend
      spec:
        containers:
        - image: bluebrown/json-server
          imagePullPolicy: Always
          name: json-server
          ports:
          - containerPort: 3000
            protocol: TCP
          resources:
            limits:
              cpu: 100m
              memory: 256Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        terminationGracePeriodSeconds: 30
  status: {}
- apiVersion: v1
  kind: Service
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"creationTimestamp":null,"labels":{"app":"backend"},"name":"backend","namespace":"default"},"spec":{"ports":[{"port":80,"protocol":"TCP","targetPort":3000}],"selector":{"app":"backend"}},"status":{"loadBalancer":{}}}
    creationTimestamp: null
    labels:
      app: backend
    name: backend
    namespace: default
  spec:
    ports:
    - port: 80
      protocol: TCP
      targetPort: 3000
    selector:
      app: backend
  status:
    loadBalancer: {}
- apiVersion: v1
  data:
    index.html: |-
      <!DOCTYPE html>
      <html lang="en">
      <head>
        <meta charset="UTF-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Hello, Kubernetes</title>
        <script type="module" defer>
          const app = document.getElementById('app');
          (async () => {
            const posts = await (await fetch('http://backend/posts')).json()
            for (const { id, title, author } of posts) {
              app.append(Object.assign(
                document.createElement('p'),
                { textContent: title },
              ))
            }
          })().catch(console.warn)
        </script>
      </head>
      <body>
        <div id="app"></div>
      </body>
      </html>
  kind: ConfigMap
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"v1","data":{"index.html":"\u003c!DOCTYPE html\u003e\n\u003chtml lang=\"en\"\u003e\n\u003chead\u003e\n  \u003cmeta charset=\"UTF-8\"\u003e\n  \u003cmeta http-equiv=\"X-UA-Compatible\" content=\"IE=edge\"\u003e\n  \u003cmeta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\"\u003e\n  \u003ctitle\u003eHello, Kubernetes\u003c/title\u003e\n  \u003cscript type=\"module\" defer\u003e\n    const app = document.getElementById('app');\n    (async () =\u003e {\n      const posts = await (await fetch('http://backend/posts')).json()\n      for (const { id, title, author } of posts) {\n        app.append(Object.assign(\n          document.createElement('p'),\n          { textContent: title },\n        ))\n      }\n    })().catch(console.warn)\n  \u003c/script\u003e\n\u003c/head\u003e\n\u003cbody\u003e\n  \u003cdiv id=\"app\"\u003e\u003c/div\u003e\n\u003c/body\u003e\n\u003c/html\u003e"},"kind":"ConfigMap","metadata":{"annotations":{},"creationTimestamp":null,"name":"frontend-data","namespace":"default"}}
    creationTimestamp: null
    name: frontend-data
    namespace: default
- apiVersion: apps/v1
  kind: Deployment
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"creationTimestamp":null,"labels":{"app":"frontend"},"name":"frontend","namespace":"default"},"spec":{"progressDeadlineSeconds":600,"replicas":1,"revisionHistoryLimit":10,"selector":{"matchLabels":{"app":"frontend"}},"strategy":{"rollingUpdate":{"maxSurge":"25%","maxUnavailable":"25%"},"type":"RollingUpdate"},"template":{"metadata":{"creationTimestamp":null,"labels":{"app":"frontend"}},"spec":{"containers":[{"image":"nginx","imagePullPolicy":"Always","name":"nginx","ports":[{"containerPort":80,"protocol":"TCP"}],"resources":{"limits":{"cpu":"100m","memory":"256Mi"}},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","volumeMounts":[{"mountPath":"/usr/share/nginx/html/","name":"frontend-data"}]}],"dnsPolicy":"ClusterFirst","restartPolicy":"Always","schedulerName":"default-scheduler","securityContext":{},"terminationGracePeriodSeconds":30,"volumes":[{"configMap":{"name":"frontend-data"},"name":"frontend-data"}]}}},"status":{}}
    creationTimestamp: null
    labels:
      app: frontend
    name: frontend
    namespace: default
  spec:
    progressDeadlineSeconds: 600
    replicas: 1
    revisionHistoryLimit: 10
    selector:
      matchLabels:
        app: frontend
    strategy:
      rollingUpdate:
        maxSurge: 25%
        maxUnavailable: 25%
      type: RollingUpdate
    template:
      metadata:
        creationTimestamp: null
        labels:
          app: frontend
      spec:
        containers:
        - image: nginx
          imagePullPolicy: Always
          name: nginx
          ports:
          - containerPort: 80
            protocol: TCP
          resources:
            limits:
              cpu: 100m
              memory: 256Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
          - mountPath: /usr/share/nginx/html/
            name: frontend-data
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        terminationGracePeriodSeconds: 30
        volumes:
        - configMap:
            name: frontend-data
          name: frontend-data
  status: {}
- apiVersion: v1
  kind: Service
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"creationTimestamp":null,"labels":{"app":"frontend"},"name":"frontend","namespace":"default"},"spec":{"ports":[{"port":80,"protocol":"TCP","targetPort":80}],"selector":{"app":"frontend"}},"status":{"loadBalancer":{}}}
    creationTimestamp: null
    labels:
      app: frontend
    name: frontend
    namespace: default
  spec:
    ports:
    - port: 80
      protocol: TCP
      targetPort: 80
    selector:
      app: frontend
  status:
    loadBalancer: {}
kind: List
metadata: {}
```

{% enddetails %}

If the output is ok, apply the resources in a development namespace

```shell
kubectl create ns dev
kubectl apply -f objects/ -n dev
kubectl get all -n dev
```

{% details Output %}

```c
NAME                            READY   STATUS             RESTARTS   AGE
pod/curler                      0/1     CrashLoopBackOff   8          21m
pod/backend-6f669dd665-fcjjx    1/1     Running            0          5s
pod/frontend-574b664bdf-4zxw6   1/1     Running            0          5s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/backend    ClusterIP   10.43.244.175   <none>        80/TCP    5s
service/frontend   ClusterIP   10.43.82.116    <none>        80/TCP    5s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend    1/1     1            1           5s
deployment.apps/frontend   1/1     1            1           5s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-6f669dd665    1         1         1       5s
replicaset.apps/frontend-574b664bdf   1         1         1       5s
```

{% enddetails %}

## Source Controlling the Project

We can now source control our project which currently looks like this.

```shell
.
├── assets
│   ├── dry-run.yaml
│   └── index.html
├── objects
│   ├── backend.deploy.yaml
│   ├── backend.svc.yaml
│   ├── frontend.configmap.yaml
│   ├── frontend.deploy.yaml
│   └── frontend.svc.yaml
└── README.md
```

Initialize a git repo and track/commit the files.

```shell
git init
git add . 
git commit -m "first commit"
```

Push it to your favorite git provider and call it infrastructure as code.

## Conclusion

By using imperative commands we can speed up the development process of the declarative yaml files and avoid many human related issues such as incorrect indentation os typos.

One the general boilerplate is generated, the files can be tweak before deployment. We can also source control the yaml files.

That way we get the best from both worlds.

### Note

The services are not published on `NodePort`or via ingress controller. This is is subject for the next topic. You can still test the pages by running curl from a pod in the same namespace.
