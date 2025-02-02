# DIA 07 - Secrets, Statefulsets, Daemonsets, Cronjobs e Operators

## Introdução

Secrets são objetos para armazenamento de informações sensiveis, como senhas ou tokens.

### Usando Secrets


#### 01 - Opaque

```
kubectl create secret generic test-secret --from-literal=username='my-app' --from-literal=password='39528$vdg7Jb'
```

```
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-secret
spec:
  containers:
  - name: envars-test-container
    image: nginx
    envFrom:
    - secretRef:
        name: test-secret
```

```
apiVersion: v1
kind: Pod
metadata:
  name: env-secretkeyref
spec:
  containers:
  - name: envars-test-container
    image: nginx
    env:
    - name: username
      valueFrom:
        secretKeyRef:
          name: test-secret
          key: username
    - name: password
      valueFrom:
        secretKeyRef:
          name: test-secret
          key: password
```

#### 02 - Dockerconfigjson

```
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```

Preencha com alguns dados fictícios e aplique no cluster. Inspecione o objeto criado para visualizar o formato do Yaml gerado.

Há duas formas de usar esse tipo de secret, diretamente no pod/deployment ou na serivceaccount.

Exemplo de uso no pod:

```
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: <your-private-image>
  imagePullSecrets:
  - name: regcred
```

Exemplo de uso na serviceaccount:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
imagePullSecrets:
- name: regcred
```

#### 03 - TLS secrets

O Kubernetes provê um tipo de secret própria para armazenar certificados TLS de aplicações (`kubernetes.io/tls`).

Crie um arquivo (`confs.cnf`) passando as configurações do certificado:

```
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no
[req_distinguished_name]
C = BR
ST = RJ
L = Rio de Janeiro
O = Sua Empresa
OU = Departamento de TI
CN = *.workshop.getup.local
[v3_req]
keyUsage = critical, digitalSignature, keyAgreement
extendedKeyUsage = serverAuth
```

Rode o openssl para a criação do certificado (`tls.crt`) e da chave privada (`tls.key`):

```
openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout tls.key -out tls.crt -config confs.cnf -sha256
```

Instale o ingress-nginx no cluster:

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

```
helm repo update
```

```
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace \
--set controller.extraArgs.default-ssl-certificate=ingress-nginx/default-certificate
```

Uma vez o ingress instalado, basta criar a secret padrão no mesmo namespace do controlador:

```
kubectl create secret tls default-certificate -n ingress-nginx\
  --cert=tls.crt \
  --key=tls.key
```

Para o teste final, crie um deployment de nginx para expô-lo através de um ingress:

```
kubectl create deploy nginx --port=80 --image=nginx:alpine
```

```
kubectl expose deploy/nginx --port=80 --target-port=80
```

```
kubectl create ingress nginx --class=nginx --rule="nginx.workshop.getup.local/=nginx:80,tls"
```

```
curl -kv https://nginx.workshop.getup.local
```

```
apiVersion: v1
kind: Secret
metadata:
  name: secret-tls
type: kubernetes.io/tls
data:
  # the data is abbreviated in this example
  tls.crt: |
        MIIC2DCCAcCgAwIBAgIBATANBgkqh ...
  tls.key: |
        MIIEpgIBAAKCAQEA7yn3bRHQ5FHMQ ...
```

Como criar a secret do tipo `kubernetes.io/tls`:

```
kubectl create secret tls my-tls-secret \
  --cert=path/to/cert/file \
  --key=path/to/key/file
```

### Usando Statefulsets

É o objeto de API Kubernetes responsável para gerenciar aplicações stateful, como bancos de dados e correlatos.

```
helm repo add openebs https://openebs.github.io/charts
helm repo update
helm install openebs -n openebs --create-namespace openebs/openebs
kubectl get storageclass
```

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "openebs-hostpath"
      resources:
        requests:
          storage: 1Gi
```


### Usando Daemonsets

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: exemplo-ds
spec:
  selector:
    matchLabels:
      app: exemplo-ds
  template:
    metadata:
      labels:
        app: exemplo-ds
    spec:
      containers:
      - image: nginx:alpine
        name: nginx
```

### Usando Jobs e Cronjobs

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.28
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

```
watch kubectl get jobs,pods,cj
```

### Usando Operators

```
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.21.2/install.sh | bash -s v0.21.2
```

```
kubectl create -f https://operatorhub.io/install/cloud-native-postgresql.yaml
```

```
apiVersion: postgresql.k8s.enterprisedb.io/v1
kind: Cluster
metadata:
  name: cluster-sample
spec:
  instances: 3
  logLevel: info
  primaryUpdateStrategy: unsupervised
  storage:
    size: 1Gi
```
