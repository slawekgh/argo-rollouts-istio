# ARGO-ROLLOUTS Z ISTIO 

Omawiamy tu mechanizm integracji argo-rollouts z traffic splittingiem w ISTIO 

Celem wprowadzenia przedstawione będzie najpierw zwykłe podejście w ISTIO do Traffic Splitting i zwykłe (proste) podstawy Argo-Rollouts 

Następnie wszystko to zostanie połączone w całość co pozwoli osiągnąć ISTIO Traffic Splitting z Argo-Rollouts

Argo-Rollouts będzie w 2 opcjach - Host-level Traffic Splitting i Subset-level Traffic Splitting



# Instalacja Argo Rollouts

## Instalacja obiektów w klastrze k8s

```
$ kubectl create namespace argo-rollouts
namespace/argo-rollouts created

$ kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
customresourcedefinition.apiextensions.k8s.io/analysisruns.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/analysistemplates.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/clusteranalysistemplates.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/experiments.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/rollouts.argoproj.io created
serviceaccount/argo-rollouts created
clusterrole.rbac.authorization.k8s.io/argo-rollouts created
clusterrole.rbac.authorization.k8s.io/argo-rollouts-aggregate-to-admin created
clusterrole.rbac.authorization.k8s.io/argo-rollouts-aggregate-to-edit created
clusterrole.rbac.authorization.k8s.io/argo-rollouts-aggregate-to-view created
clusterrolebinding.rbac.authorization.k8s.io/argo-rollouts created
configmap/argo-rollouts-config created
secret/argo-rollouts-notification-secret created
service/argo-rollouts-metrics created
deployment.apps/argo-rollouts created
```


## Instalacja kubectl-plugina 

```
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-darwin-amd64
sudo mv ./kubectl-argo-rollouts-darwin-amd64 /usr/local/bin/kubectl-argo-rollouts
```



# ISTIO bez Argo Rollouts 

Zasady działania klasycznego Traffic Splitting w ISTIO 

w katalogu ISTIO-BASE tego repozytorium są odpowiednie YAMLe do wdrożenia:

na klastrze k8s gdzie mamy postawione ISTIO tworzymy dowolny namespace 

```
kubectl create namespace test-istio
kubectl config set-context --current --namespace=test-istio
```

nastepnie labelujemy (tu 3 przykłady z różnych środowisk) ten namespace aby działał autoinject dla envoy-proxies:
```
kubectl label namespace test-istio istio-injection=enabled --overwrite
kubectl label namespace test-istio istio-injection- istio.io/rev=asm-182-2 --overwrite
kubectl label namespace test-istio istio-injection- istio.io/rev=asm-1162-2 --overwrite
```

i wgrywamy wszystkie obiekty do testu :

```
kubectl apply -f deploy-consumer.yaml
kubectl apply -f deploy-server-v1.yaml
kubectl apply -f deploy-server-v2.yaml
kubectl apply -f service_clusterip-consumer.yaml
kubectl apply -f service_clusterip-Server-v1v2.yaml
kubectl apply -f virtual-service-Server-wagi.yml
kubectl apply -f destination-rule-Server-v1-v2.yml
```

zmiany w Traffic-Splitting realizujemy poprzez edycję virtual-service-Server-wagi.yml

```
  http:
  - route:
    - destination:
        host: app07.slawek.svc.cluster.local
        subset: version-v2
      weight: 100
    - destination:
        host: app07.slawek.svc.cluster.local
        subset: version-v1
      weight: 0
 ```

Mechanizm w skrócie polega na tym że są 2 x deploy-Server , jeden poza label name:app07 ma też label version v1 , drugi ma v2 

DestinationRule (dla tego deployu SERVER) ma zdefiniowane 2 subsety - jeden subset żyje na labels:version:v1 a drugi na v2 

z kolei VirtualService (dla tego deployu SERVER) ma 1 route a w niej 2 destinations - jedna na subset1 a druga na subset2

Administrator kręci wagami wstawiając 100_do_0, 5_do_95 itp 

Powyższe to podstawy istiowych podstaw ale trzeba to mieć przed oczami przy analizie modeli opartych na Argo-Rollout








# ARGO ROLLOUTS bez istio - BASICS

Zasady działania AR , chwilowo bez istio 

To również podstawy podstaw AR ale trzeba je zrozumieć żeby potem łączyć AR+ISTIO 


*A Rollout is Kubernetes workload resource which is equivalent to a Kubernetes Deployment object. It is intended to replace a Deployment object in scenarios when more advanced deployment or progressive delivery functionality is needed*


Katalog z plikami YAML w tym repo: AR-bez-ISTIO

Tworzymy namespace i obiekty w nim:

```
$ kk create ns test-ar
namespace/test-ar created

$ kk -n test-ar apply -f rollout-deploy-server-v1.yaml
rollout.argoproj.io/test-rollout created

$ kk -n test-ar apply -f service_clusterip-Server-v1v2.yaml
service/app07 created

$ kk -n test-ar apply -f deploy-consumer.yaml   #ten deploy nie należy do zestawu, tworzymy go dla wygody żeby mieć obok PODa z curlem 
deployment.apps/consumer created

$ kk -n test-ar get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/test-rollout-746b5594b8-5mspv   1/1     Running   0          112s
pod/test-rollout-746b5594b8-5rp7k   1/1     Running   0          112s
pod/test-rollout-746b5594b8-6qbn4   1/1     Running   0          112s
pod/test-rollout-746b5594b8-rbrn4   1/1     Running   0          112s
pod/test-rollout-746b5594b8-wck6b   1/1     Running   0          112spod/consumer-58f6fd4c95-lzcvg      1/1     Running   0          40s

NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/app07   ClusterIP   10.108.8.117   <none>        8080/TCP   3s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/test-rollout-746b5594b8   5         5         5       113s
```


zmieniamy w naszym rollout obraz  

```
$ kubectl argo rollouts set image test-rollout app07=gimboo/nginx_nonroot2 -n test-ar
rollout "test-rollout" image updated

$ kubectl argo rollouts get rollout test-rollout -n test-ar
Name:            test-rollout
Namespace:       test-ar
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/8
  SetWeight:     20
  ActualWeight:  20
Images:          gimboo/nginx_nonroot (stable)
                 gimboo/nginx_nonroot2 (canary)
Replicas:
  Desired:       5
  Current:       5
  Updated:       1
  Ready:         5
  Available:     5

NAME                                      KIND        STATUS     AGE  INFO
⟳ test-rollout                            Rollout     ॥ Paused   13m  
├──# revision:2                                                      
│  └──⧉ test-rollout-dd4cc6cb4            ReplicaSet    Healthy  69s  canary
│     └──□ test-rollout-dd4cc6cb4-vrd4x   Pod           Running  69s  ready:1/1
└──# revision:1                                                      
   └──⧉ test-rollout-746b5594b8           ReplicaSet    Healthy  13m  stable
      ├──□ test-rollout-746b5594b8-5mspv  Pod           Running  13m  ready:1/1
      ├──□ test-rollout-746b5594b8-5rp7k  Pod           Running  13m  ready:1/1
      ├──□ test-rollout-746b5594b8-rbrn4  Pod           Running  13m  ready:1/1
      └──□ test-rollout-746b5594b8-wck6b  Pod           Running  13m  ready:1/1

```


Jak widać AR zblokował się na  20% (tak jest w definicji) i czeka na dalsze kroki z naszej strony 


*When the demo rollout reaches the second step, we can see from the plugin that the Rollout is in a paused state, and now has 1 of 5 replicas running the new version of the pod template, and 4 of 5 replicas running the old version. This equates to the 20% canary weight as defined by the setWeight: 20 step.*


```
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {}
      - setWeight: 40
      - pause: {duration: 10}
      - setWeight: 60
      - pause: {duration: 10}
      - setWeight: 80
      - pause: {duration: 10}
```

aby pchnąć wdrożenie do przodu trzeba wykonać promote, aby cofnąć należy wykonać abort:


```
kubectl argo rollouts promote test-rollout -n test-ar
kubectl argo rollouts abort test-rollout -n test-ar
```

Tu ważna uwaga - po cofnięciu mimo że stan jest ok/stabilny to i tak wpadamy w stan Degraded (no bo degraded bo framework zakłada że się nie udało - zalegalizować można robiąc set na stary-stabilny konf)

testy oczywiście realizujemy via POD consumera:

```
$ kk -n test-ar exec -ti consumer-58f6fd4c95-lzcvg bash
nginx@consumer-58f6fd4c95-lzcvg:/$ curl app07:8080
test-rollout-dd4cc6cb4-mmc2s
<br>2
nginx@consumer-58f6fd4c95-lzcvg:/$ curl app07:8080
test-rollout-dd4cc6cb4-88nhd
<br>2
```

i to tyle podstawowej teorii o AR 



# ARGO ROLLOUTS Z ISTIO 


Zacznijmy od tego że Argo Rollout w kontekście ISTIO dostarcza 2 odrębne podejścia: 

*Istio provides two approaches for weighted traffic splitting, both approaches are available as options in Argo Rollouts:*

*Host-level traffic splitting*

*Subset-level traffic splitting*



## Subset-level 


Subset-level jest bliższy klasycznemu podejściu ISTIO-way więc od niego zaczniemy

https://argo-rollouts.readthedocs.io/en/stable/features/traffic-management/istio/#subset-level-traffic-splitting

Folder w tym repo z plikami YAML:  AR-z-ISTIO-SubsetLevelTrafficSplitting


tworzymy osobny NS, labelujemy go celem istio-injection i zakładamy konieczne obiekty:

```
kubectl create ns test-ar-istio
kubectl config set-context --current --namespace=test-ar-istio
kubectl -n istio-system get pods -l app=istiod --show-labels | grep rev
kubectl label namespace test-ar-istio istio-injection- istio.io/rev=asm-1162-2 --overwrite
kubectl apply -f deploy-consumer.yaml
kubectl apply -f destination-rule-Server-v1-v2.yaml
kubectl apply -f rollout-deploy-server-ISTIO.yaml
kubectl apply -f service_clusterip-consumer.yaml
kubectl apply -f service_clusterip-Server-v1v2.yaml
kubectl apply -f virtual-service-Server-wagi.yaml
```

teraz można wykonywac operacje wkoło AR:

```
kubectl argo rollouts set image test-rollout-istio app07=gimboo/nginx_nonroot3
kubectl argo rollouts promote test-rollout-istio
kubectl argo rollouts abort test-rollout-istioi
```

podczas testów widać że po SET wagi ustawiane są na 5 do 95 :

```
$ kk get vs
NAME                GATEWAYS                             HOSTS                                                          AGE
virtservice-app07   ["mesh","test-ar-istio/mygateway"]   ["app07.test-ar-istio.svc.cluster.local","uk.bookinfo7.com"]   9m41s 

$ kk get vs virtservice-app07 -o yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: virtservice-app07
  namespace: test-ar-istio
spec:
  gateways:
  - mesh
  - test-ar-istio/mygateway
  hosts:
  - app07.test-ar-istio.svc.cluster.local
  - uk.bookinfo7.com
  http:
  - name: primary
    route:
    - destination:
        host: app07.test-ar-istio.svc.cluster.local
        subset: version-v2
      weight: 5
    - destination:
        host: app07.test-ar-istio.svc.cluster.local
        subset: version-v1
      weight: 95
```

zaś po PROMOTE na 100 do 0 

```
$ kk get vs virtservice-app07 -o yaml[...] 
 http:
  - name: primary
    route:
    - destination:
        host: app07.test-ar-istio.svc.cluster.local
        subset: version-v2
      weight: 0
    - destination:
        host: app07.test-ar-istio.svc.cluster.local
        subset: version-v1
      weight: 100
```

dla przypomnienia w oryginalnym wsadowym pliku dla VS te wartości były zupełnie inne, ale nie ma to już znaczenia bo to AR zarządza od tej pory tymi wagami - 
innymi słowy modyfikuje nasze obiekty bez naszej wiedzy: 

```
$ kk get  vs  -o yaml | grep -v apiVer| grep weight
        weight: 100
        weight: 0

$ kubectl argo rollouts set image test-rollout-istio app07=gimboo/nginx_nonroot2
rollout "test-rollout-istio" image updated

$ kk get  vs  -o yaml | grep -v apiVer| grep weight
        weight: 95
        weight: 5

$ kubectl argo rollouts promote test-rollout-istio
rollout 'test-rollout-istio' promoted

$ kk get  vs  -o yaml | grep -v apiVer| grep weight
        weight: 100
        weight: 0
```

to 95/0 bierze się oczwyiście stąd:

```
kind: Rollout
[...]      steps:
      - setWeight: 5

```





Dla przypomnienia - w podejściu klasycznym (bez ARGO-rollout tylko czyste ważenie na istio destination-rules/subsets) mechanizm jest następujący:

1. są 2 x deploy-Server , jeden poza label name:app07 ma też label version v1 , drugi ma v2 
2. jest destination-rule (dla tego deployu SERVER) która ma zdefiniowane 2 subsety - jeden subset żyje na labels:version:v1 a drugi na v2 i
3. z kolei VirtualService (dla tego deployu SERVER) ma 1 route a w niej 2 destinations - jedna na subset1 a druga na subset2, jedna ma weight 100 a druga weight 0

w skrócie:

```
$ cat /virtual-service-Server-wagi.yml
[...]
  - route:
    - destination:
        host: app07.slawek.svc.cluster.local
        subset: version-v2
      weight: 100
    - destination:
        host: app07.slawek.svc.cluster.local
        subset: version-v1
      weight: 0

$ cat /destination-rule-Server-v1-v2.yml
[...]
  subsets:
  - labels:
      version: v1
    name: version-v1
  - labels:
      version: v2
    name: version-v2
```


Tu z kolei różnic między plikami VS i DestinationRule nie ma (pomijając że dla wersji z AR trzeba w DestRule wstawić jakiś label generyczny (np name=app07) 
+jest jedna drobna bo w VS jak jest route to tu ma name=primary) nie ma znaczenia jak ustawimy wagi w VS.yaml bo AR i tak zaraz przejmie sprawy 



po wgraniu naszego destination-rule zostaje ona natychmiast zmodyfikowana przez AR:


```
$ kubectl get destinationrules destrule-app07 -o yamlapiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: destrule-app07
  namespace: test-ar-istio
spec:
  host: app07.test-ar-istio.svc.cluster.local
  subsets:
  - labels:
      name: app07
      rollouts-pod-template-hash: 5bfbf9599
    name: version-v1
  - labels:
      name: app07
      rollouts-pod-template-hash: 5bfbf9599
    name: version-v2



```

i w takim classic-trybie-istio (gdzie mamy 2 osobne deploye - jeden ma labels na PODach version: v1 , drugi ma v2 ) to nasza DestinationRule jest oparta o te labele version-v1-v2tu w AR zaszła zmiana - skoro AR modyfikuje za naszymi plecami naszą dest-rule (dodając jej też extra labele rollouts-pod-template-hash) to wlaśnie na tej labeli różnicowane są PODy 
$ kk get po --show-labels
NAME                                  READY   STATUS    RESTARTS   AGE   LABELS
test-rollout-istio-5bfbf9599-dbcd6    2/2     Running   0          21m   name=app07,rollouts-pod-template-hash=5bfbf9599,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=test-rollout-istio-5bfbf9599,service.istio.io/canonical-revision=latest,topology.istio.io/network=a-r-02-default
test-rollout-istio-746b5594b8-pv4wr   2/2     Running   0          11m   name=app07,rollouts-pod-template-hash=746b5594b8,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=test-rollout-istio-746b5594b8,service.istio.io/canonical-revision=latest,topology.istio.io/network=a-r-02-default



