### Moon installation
How to install Minikube without VM and Moon

## Requirements: 
  * 2 cpus minimum;
  * free ports 4444 and 8080

## How to install [Minikube](https://github.com/kubernetes/minikube):
```shell
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube
curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && chmod +x kubectl

export MINIKUBE_WANTUPDATENOTIFICATION=false
export MINIKUBE_WANTREPORTERRORPROMPT=false
export MINIKUBE_HOME=$HOME
export CHANGE_MINIKUBE_NONE_USER=true
mkdir $HOME/.kube || true
touch $HOME/.kube/config

export KUBECONFIG=$HOME/.kube/config
sudo -E ./minikube cpus 2 start --vm-driver=none

# After these steps Running Kubernetes cluster and 
# kubectl client installed and pointing to the cluster
# will be avaible
```

## Useful commands for minikube:
   * `./minikube version`
   * `./minikube status`
   * `./minikube dashboard` - it will be accessible outside at [IP of machine where Minikube installed]:30000), but sometimes you have to wait a couple of seconds
 
 ## Useful commands for kubectl: 
    * `kubectl get nodes`
    * `kubectl get pods`
    * `kubectl get services`
    * `kubectl get deployments`
    * `kubectl delete deployment [name of the deployment]`
    * `kubectl delete service [name of the service]`
    * `kubectl describe service [name of the service]`
     etc

## How to install [Moon](http://aerokube.com/moon/latest/):
```shell
$ htpasswd -Bbn test test-password >> users.htpasswd
$ kubectl create secret generic users --from-file=./users.htpasswd

$ mkdir -p quota
$ touch quota/browsers.json && nano quota/browsers.json 
```

# [browsers.json](https://github.com/aerokube/moon/blob/gh-pages/latest/files/browsers.json):

```shell
{
  "firefox": {
    "default": "57.0",
    "versions": {
      "57.0": {
        "image": "selenoid/vnc:firefox_57.0",
        "port": "4444",
        "path": "/wd/hub"
        }
    }
  },
"chrome": {
    "default": "62.0",
    "versions": {
      "62.0": {
        "image": "selenoid/vnc:chrome_62.0",
        "port": "4444"
        }
    }
  }
}
```

```shell
$ touch moon-sessions.yaml && nano moon-sessions.yaml
```

# [moon-sessions.yaml](https://github.com/aerokube/moon/blob/gh-pages/latest/files/moon-sessions.yaml):

```shell
apiVersion: v1
kind: ResourceQuota
metadata:
  name: max-moon-sessions
spec:
  hard:
    pods: "8"
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: moon-rbac
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

```shell
$ touch moon.yaml && nano moon.yaml
```

# [moon.yaml](https://github.com/aerokube/moon/blob/gh-pages/latest/files/moon.yaml): 
```shell
kind: Service
apiVersion: v1
metadata:
  name: moon
spec:
  selector:
    app: moon
  ports:
  - protocol: TCP
    port: 4444
  type: NodePort
---
apiVersion: apps/v1beta1 
kind: Deployment
metadata:
  name: moon
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: moon
    spec:
      containers:
      - name: moon
        image: aerokube/moon:latest
        resources:
          limits:
            cpu: "1"
            memory: "512Mi"
          requests:
            cpu: "0.25"
            memory: "64Mi"
        ports:
        - containerPort: 4444
        volumeMounts:
        - name: quota
          mountPath: /quota
          readOnly: true
        - name: users
          mountPath: /users
          readOnly: true
      volumes:
      - name: quota
        configMap:
          name: quota
      - name: users
        secret:
          secretName: users
```

```shell
$ touch moon-api.yaml && nano moon-api.yaml
```

# [moon-api.yaml](https://github.com/aerokube/moon/blob/gh-pages/latest/files/moon-api.yaml):
```shell
kind: Service
apiVersion: v1
metadata:
  name: moon-api
spec:
  selector:
    app: moon-api
  ports:
  - protocol: TCP
    port: 8080
  type: NodePort
---
apiVersion: apps/v1beta1 
kind: Deployment
metadata:
  name: moon-api
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: moon-api
    spec:
      containers:
      - name: moon-api
        image: aerokube/moon-api:latest
        resources:
          limits:
            cpu: "0.25"
            memory: "128Mi"
          requests:
            cpu: "0.1"
            memory: "64Mi"        
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: quota
          mountPath: /quota
      volumes:
      - name: quota
        configMap:
          name: quota
      - name: users
        secret:
          secretName: users      
```

```shell
$ kubectl create configmap quota --from-file=quota
$ kubectl create -f moon-sessions.yaml
$ kubectl create -f moon.yaml
$ kubectl create -f moon-api.yaml
```

## How to determine IP addresses or hostnames for moon and moon-api services. 
When testing in Minikube this can be done with the following commands:

```shell
$ ./minikube service moon --url
```

```shell
$ ./minikube service moon-api --url
```

## Run your tests against moon service like you do with regular Selenium:
`http://[YourIP]:[Your Moon hostname]/wd/hub`

## Check that moon-api returns statistics:
`curl -s http://[YourIP]:[Your Moon hostname]/status`

## How to uninstall Minikube: 
```shell
$ ./minikube stop
$ ./minikube delete
$ sudo rm -rf ~/.minikube
```

## How to delete Moon or Moon-api service and deployment:
`$ kubectl delete service, deployment moon`

## How to update moon-sessions: 
`kubectl replace -f moon-sessions.yaml`
