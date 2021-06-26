# kind-image-locality-demo

## requirements
- kind    `0.11.0`
- docker  `20.10.7`
- kubectl `1.17.3`

# intro

Naszym zadaniem będzie przetestowanie możliwości własnej konfiguracji polityki schedulera - poda z przestrzeni nazw `kube-system`, którego zadaniem jest dopisanie do nowo utworzonego manifestu poda informacji o tym na jakim nodzie ma zostać wykonany. W przypadku braku tego klucza pod będzie ciągle w stanie `Pending`.

```diff
spec:
  containers:
  (...)
+  nodeName: scheduler-worker2
+  schedulerName: default-scheduler
```

W przypadku manifestu w którym brakuje etykiety `schedulerName` domyślnie zostanie przypisany systemowy scheduler - `default-scheduler`.

Zadaniem schedulera jest na podstawie danych zebranych z nodów, dodać etykiete `nodeName` - zawierającą informację na którym nodzie ma wykonać się pod.

### kryteria schedulera

Domyślnie scheduler oblicza score każdego noda na podstawie następujących [kryteriów](https://kubernetes.io/docs/reference/scheduling/policies/), jednak my zajmiemy się jednym z nich a mianowicie:

> **ImageLocalityPriority**: faworyzuje nody, które mają lokalnie zaciągnięte obrazy kontenerów


### manifesty podów systemowych
Manifesty podów systemowych znajdują się w katalogu `/etc/kubernetes/manifests` na master nodach

`kind` umożliwia edycję tego manifestu za pomocą definicji manifestu klastra

```diff
$ cat manifests/cluster-config.yaml
(...)
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: ClusterConfiguration
        scheduler:
          extraArgs:
+            config: /etc/kubernetes/scheduler-config.conf
+            v: '15'
          extraVolumes:
            - name: configuration
              hostPath: /etc/kubernetes/scheduler-config.conf
              mountPath: /etc/kubernetes/scheduler-config.conf
              readOnly: true
              pathType: File
(...)
```

Należy tutaj zwrócić uwagę na dwa pola:
 - `v`(erbose) - w celu wyświetlenia wartości score dla poda należy zwięszkyć poziom logowania `kube-schedulera` do 15
 - `config` - scieżka do manifestu zawierającego zasób `KubeSchedulerConfiguration`, który umożliwi nam konfiguracje polityki schedulowania

### KubeSchedulerConfiguration
 KubeSchedulerConfiguration jest abstrakcyjnym zasobem, umożliwiającym kontrolę kryteriów algorytmu liczącego score dla Noda. W naszym przypadku chcemy wyłączyć wszystkie dyśle kryteria oraz pozostawić jedynie `ImageLocality` z wysokim współczynnikiem

```diff
kind: KubeSchedulerConfiguration
percentageOfNodesToScore: 100
profiles:
  - schedulerName: default-scheduler
    plugins:
      score:
+       disabled:
+         - name: '*'
+       enabled:
+         - name: ImageLocality
+           weight: 1000
```

# tutorial

Poniższe ćwiczenie będziemy przeprowadzać na [kindzie](https://kind.sigs.k8s.io/docs/user/quick-start/). Jest to klaster kubernetesa bazujący na obrazach dockerowych.

Aby utworzyć nowy klaster wykonaj:

```zsh
kind create cluster --name scheduler --config=cluster-config.yml
```

Poprawnie stworzony klaster powinien dać następujący output:

```zsh
$ kind get clusters
scheduler
$ kubectl get nodes -o wide
NAME                      STATUS   ROLES                  AGE   VERSION   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
scheduler-control-plane   Ready    control-plane,master   32m   v1.21.1   Ubuntu 20.10   5.8.0-59-generic   containerd://1.5.1
scheduler-worker          Ready    <none>                 31m   v1.21.1   Ubuntu 20.10   5.8.0-59-generic   containerd://1.5.1
scheduler-worker2         Ready    <none>                 31m   v1.21.1   Ubuntu 20.10   5.8.0-59-generic   containerd://1.5.1
scheduler-worker3         Ready    <none>                 31m   v1.21.1   Ubuntu 20.10   5.8.0-59-generic   containerd://1.5.1
```

W tej konfiguracji stworzony klaster posiada tylko jednego master noda o nazwie `scheduler-control-plane` oraz 3 worker node'y. Warto też zauważyć że w przypadku tego setupu naszym runtime'em kontenerowym jest `containerd` a nie `dockerd` - rzutować to będzie na drobne zmiany m.in: socket do komunikacji z demonek w kernelu znajduje się pod ścieżką `/run/containerd/containerd.sock` a z poziomu cli będziemy wchodzić z interakcję za pomocą komendy `crictl`.

W celu wypisania wszystkich obrazów znajdujących się w kontenerze worker nodzie klastra - `schduler-worker` należy wykonac polecenie:
```zsh
$ docker exec -it scheduler-worker crictl images
IMAGE                                      TAG                  IMAGE ID            SIZE
docker.io/kindest/kindnetd                 v20210326-1e038dc5   6de166512aa22       54MB
docker.io/rancher/local-path-provisioner   v0.0.14              e422121c9c5f9       13.4MB
k8s.gcr.io/build-image/debian-base         v2.1.0               c7c6c86897b63       21.1MB
k8s.gcr.io/coredns/coredns                 v1.8.0               296a6d5035e2d       12.9MB
k8s.gcr.io/etcd                            3.4.13-0             0369cf4303ffd       86.7MB
k8s.gcr.io/kube-apiserver                  v1.21.1              6401e478dcc01       127MB
k8s.gcr.io/kube-controller-manager         v1.21.1              d0d10a483067a       121MB
k8s.gcr.io/kube-proxy                      v1.21.1              ebd41ad8710f9       133MB
k8s.gcr.io/kube-scheduler                  v1.21.1              7813cf876a0d4       51.9MB
k8s.gcr.io/pause                           3.4.1                0f8457a4c2eca       301kB
```

Następnie należy przekopiwać do mastera naszą nową konfigurację schedulera

```zsh
docker cp scheduler-config.conf scheduler-control-plane:/etc/kubernetes/scheduler-config.conf
```

a następnie w celu zaczytania nowej konfiguracji należy usunąć obecnego schedulera:

```zsh
kubectl delete pod kube-scheduler-scheduler-control-plane -n kube-system 
```

w celu zobaczenia logów dotyczących schedulera należy konać polecenie:

```zsh
kubectl logs kube-scheduler-scheduler-control-plane -n kube-system -f
```

W celu przetestowania działania obecnej konfiguracji należy kilkukrotnie utworzyć poda, bazującego na obrazie `busybox` edytując za każdym razem jego nazwę (tak, aby utowrzyć fizycznie kilka Podów)

```zsh
$ kubectl apply -f manifests/busybox-pod.yaml
pod/busybox-sleep-0 changed
```
następnie edytować pole `metadata.name` do busybox-sleep-11100 i operację powtórzyć kilkukrotnie

Aby docelowo utworzyć kilka podów z tego samego obrazu: 
```zsh
$ kubectl get pods -o wide
NAME                  READY   STATUS    RESTARTS   AGE   IP           NODE                NOMINATED NODE
busybox-sleep-0       1/1     Running   0          67m   10.244.2.3   scheduler-worker2   <none> 
busybox-sleep-1110    1/1     Running   0          67m   10.244.2.2   scheduler-worker2   <none> 
busybox-sleep-11100   1/1     Running   0          13s   10.244.3.3   scheduler-worker3   <none> 
```

Oraz weryfikuję czy dany obraz znajduje się na właścicywch nodach:
```zsh
$ docker exec -it scheduler-worker2 crictl images | grep busybox
docker.io/library/busybox                  latest               69593048aa3ac       771kB
$ docker exec -it scheduler-worker3 crictl images | grep busybox
docker.io/library/busybox                  latest               69593048aa3ac       771kB
$ docker exec -it scheduler-worker crictl images | grep busybox 
```

W poniższym zrzucie ekranu widać że scheduler zaczytał poprawnie nową konfigurację - w którym jedyną metryką jest `ImageLocality` jednak liczy dla niego niepoprawny score - dla każdego noda jest równy 0
![](img/score.png)

Powodem mogą być następujące czynniki
- Za zbieranie informacji o lokalnych obrazach na worker nodach opowiada `kubelet`. W przypadku implementacji `kinda` jest w uboższej konfiguracji - nie posiada tej funkcjonalności. Argumentem za takim scenariuszej jest fakt, iż kontenery są to fizczynie procesy, które współdzielą ze sobą kernel - w którym znajduje się tylko jeden runtime kontenerowy. Fizczynie `containerd` nie jest instalowany wewnątrz każdego kontenerea, tylko jest strzykiwany poprzez `socket` - `/run/containerd/containerd.sock` w który wchodzi w interakcję z właściwym demonem z przestrzeni jądra.


# next steps

W zaistniałej sytuacji chcemy zaproponować następujący workaround - DemonSet, który będzie cykliczne monitorował stan kontenerów `kinda` oraz logowanie tych informacji do `etcd`

Przykładowy manifest:

```diff
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    name: rancher-crictl
  name: rancher-crictl
spec:
  selector:
    matchLabels:
      name: rancher-crictl
  template:
    metadata:
      labels:
        name: rancher-crictl
    spec:
      volumes:
        - name: cri-sock
          hostPath:
            path: "/run/containerd/containerd.sock"
      containers:
        - name: rancher-crictl
          image: rancher/crictl:v1.19.0
          command: [ "/bin/bash", "-c", "--" ]
+         args: [ "while true; do crictl images ls; done;" ]
          securityContext:
            privileged: true
          volumeMounts:
+           - mountPath: /run/containerd/containerd.sock
              name: cri-sock
              readOnly: false
```

Zwrócić w nim należy na dwie ważne sekcje - przekazanie do wnętrza kontenera socketu z worker node'a oraz cyklicznie wypisywanie na stdout listy wszystkich kontenerów.