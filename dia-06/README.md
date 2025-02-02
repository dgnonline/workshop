# DIA 06 - Volumes

## Introdução

### Usando Volumes

#### 01 - Emptydir

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: exemplo-01
spec:
  replicas: 1
  selector:
    matchLabels:
      app: emptydir
  template:
    metadata:
      labels:
        app: emptydir
    spec:
      containers:
      - name: escrita
        image: k8s.gcr.io/busybox
        args:
        - /bin/sh
        - -c
        - while sleep 30; do touch /escrita/$(date +%H)-$(date +%M)-$(date +%S); done
        volumeMounts:
        - mountPath: /escrita
          name: pasta-comum
      - name: leitura
        image: k8s.gcr.io/busybox
        args:
        - /bin/sh
        - -c
        - while sleep 30; do ls -l /leitura; done
        volumeMounts:
        - mountPath: /leitura
          name: pasta-comum
      volumes:
      - name: pasta-comum
        emptyDir: {}
```

Veja a saída de log do container `leitura`. Derrube o Pod e faça uma nova conferência. Note que as informação

#### 02 - Hostpath

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: exemplo-02
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hostpath
  template:
    metadata:
      labels:
        app: hostpath
    spec:
      containers:
      - name: escrita
        image: k8s.gcr.io/busybox
        args:
        - /bin/sh
        - -c
        - while sleep 30; do touch /app/$(date +%H)-$(date +%M)-$(date +%S); done
        volumeMounts:
        - mountPath: /app
          name: temp
      volumes:
      - name: temp
        hostPath:
          path: /tmp
```

Aguarde o pod rodar pelo menos por um minuto. Verifique a presença dos arquivos gerados na pasta `/app` do pod. Mate o pod e faça o mesmo procedimento para o seguinte. Note que os arquivos gerados pelo primeiro Pod foram persistidos e montados no segundo.


#### 03 - PV e PVC - Provisionamento estático

PV:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-exemplo-03
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

PVC:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-exemplo-03
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

POD:

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-exemplo-03
spec:
  volumes:
    - name: pv
      persistentVolumeClaim:
        claimName: pvc-exemplo-03
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv
```

Crie um arquivo com sob o nome index.html e adicione seu nome internamente.

```
kubectl exec -it pod-exemplo-03 -- bash
echo SEU_NOME >> /usr/share/nginx/html/index.html
curl http://localhost
```

Mate o Pod e rode o comando `curl` novamente para checar se o arquivo criado na etapa anterior foi persistido em disco.

#### 04 - PV e PVC - Provisionamento dinâmico

Para esse cenário dinâmico, o administrador precisa configurar um `storageClass` para permitir que a API gerencie e crie os volumes automaticamente.

Instale o Openebs com Helm para permitir o uso de um StorageClass no seu cluster:

```
helm repo add openebs https://openebs.github.io/charts
helm repo update
helm install openebs -n openebs --create-namespace openebs/openebs
kubectl get storageclass
```

Instale o PVC no cluster:

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-exemplo-04
spec:
  storageClassName: openebs-hostpath
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1G
```

Instale o Pod no cluster:

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-exemplo-04
spec:
  volumes:
    - name: local-storage
      persistentVolumeClaim:
        claimName: pvc-exemplo-04
  containers:
    - name: hello-container
      image: busybox
      command:
        - sh
        - -c
        - 'while true; do echo "`date` [`hostname`] Hello from OpenEBS Local PV." >> /mnt/store/greet.txt; sleep $(($RANDOM % 5 + 300)); done'
      volumeMounts:
        - mountPath: /mnt/store
          name: local-storage
```

Identifique o PV criado e localize o arquivo criado pelo pod.

#### 05 - Variáveis de ambiente

```
apiVersion: v1
kind: Pod
metadata:
  name: envar-demo
  labels:
    purpose: demonstrate-envars
spec:
  containers:
  - name: envar-demo-container
    image: gcr.io/google-samples/node-hello:1.0
    env:
    - name: DEMO_GREETING
      value: "Hello from the environment"
    - name: DEMO_FAREWELL
      value: "Such a sweet sorrow"
```

```
kubectl exec -it envar-demo -- env
```

#### 06 - ConfigMaps

É um objeto de API usado para armazenamento de informações não confidenciais no padrão de chave valor. Pods podem consumir esses objetos como variáveis de ambiente, argumentos de linha de comando ou como arquivos de configuração em um volume.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # file-like keys
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```

```
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # Define the environment variable
        - name: PLAYER_INITIAL_LIVES # Notice that the case is different here
                                     # from the key name in the ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: game-demo           # The ConfigMap this value comes from.
              key: player_initial_lives # The key to fetch.
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # You set volumes at the Pod level, then mount them into containers inside that Pod
    - name: config
      configMap:
        # Provide the name of the ConfigMap you want to mount.
        name: game-demo
        # An array of keys from the ConfigMap to create as files
        items:
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"
```
