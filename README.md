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



po wgraniu naszego destination-rule zostaje ona natychmiast zmodyfikowana przez AR (patrzymy na rollouts-pod-template-hash):


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

Co ważne - w classic-trybie-istio (gdzie mamy 2 osobne deploye - jeden ma labels na PODach version: v1 , drugi ma v2 ) to nasza DestinationRule jest oparta o te labele version-v1-v2i

tu w AR zaszła zmiana - skoro AR modyfikuje za naszymi plecami naszą dest-rule (dodając jej też extra labele rollouts-pod-template-hash) to wlaśnie na tej labeli różnicowane są PODy:


```
$ kk get po --show-labels
NAME                                  READY   STATUS    RESTARTS   AGE   LABELS
test-rollout-istio-5bfbf9599-dbcd6    2/2     Running   0          21m   name=app07,rollouts-pod-template-hash=5bfbf9599,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=test-rollout-istio-5bfbf9599,service.istio.io/canonical-revision=latest,topology.istio.io/network=a-r-02-default
test-rollout-istio-746b5594b8-pv4wr   2/2     Running   0          11m   name=app07,rollouts-pod-template-hash=746b5594b8,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=test-rollout-istio-746b5594b8,service.istio.io/canonical-revision=latest,topology.istio.io/network=a-r-02-default
```



## Jednym słowem proces dla Traffic-Splitting wygląda nastepująco:

### Dla wersji z gołym ISTIO:

 - administrator powołuje 2 deploymenty + przykrywający je k8s-svc. (każdy Deploy ma inny label version=v1/v2) 
 - do tego dodaje DestinationRule która definiuje 2 subsety bazujące na tych labelach z version
 - do tego na koniec dodaje VirtualService w którym w .route.destination odwołuje się do tych 2 subsetów i ma wstawione w te 2 miejsca wstępne wagi
 - admin zaczyna administrować TrafficSplitingiem poprzez edycję tego VS wsadzając wagi 100/0, 90/10 itd 


### Dla wersji z ArgoRollouts-ISTIO 
 - administrator zamiast 2xDeploy powołuje ROLLOUT w którym: 
     - wskazuje na VirtService którym AR będzie kręcił via wstawianie wag (dlatego podaje się tam nazwę route - np "primary") 
     - wskazuje na DestRule (podaje dla AR jej nazwę, canarySubsetName version=v2 , stableSubsetName version=v1) 
     - określa steps dla promocji rolloutu oraz pause{} 
     - emuluje Deploy (wskazuje image, resources itd) 
 - admin powołuje DestRule (yaml wsadowy (czyli ten z day0) jest inny niż dla gołego ISTIO, nie ma różnicowania, obydwa subsety muszą byc zrobione na czymś co jest takie same dla nich dwóch - potem AR natychmiast i w trybie ciągłym zmienia mu subsety ktore bazują od teraz _również_ na labelach rollouts-pod-template-hash=RANDOM_Number)
 - admin powołuje VirtService (również identyczny jak dla scenariusza z gołym ISTIO) ktorym już za chwilę AR będzie zarządzał - ale jedynie kręcąc w nim wagami 
 - to nie admin ale tym razem AR zaczyna zarządzać TraficSplitingiem poprzez ustawianie wag w VirtService - 100/0 i 95/5 ORAZ poprzez wstawanie do subsetów w naszej DestinationRule warunków opartych na rollouts-pod-template-hash

Wróćmy do zmian w rolloutach - do tej pory zmienialiśmy jedynie obraz (via kubectl argo rollouts set image test-rollout-istio app07=gimboo/nginx_nonroot3)


sprawdźmy jak to działa ale z podmianą CM, a zatem wprowadźmy nowy rollout zawierający odwołanie do CM + nowy obiekt CM

```
rollout-deploy-server-ISTIO-ConfigMap.yaml
config-map-01.yaml
```

różnica w kontekście deploymentu (a właściwie oczywiście jego emulacji) jest następująca:

```
$ diff rollout-deploy-server-ISTIO.yaml rollout-deploy-server-ISTIO-ConfigMap.yaml.optional
35c35,43
<
---
>         volumeMounts:
>         - name: config
>           mountPath: "/config"
>           readOnly: true
>       volumes:
>       - name: config
>         configMap:
>           # Provide the name of the ConfigMap you want to mount.
>           name: cm-01
```

dodajemy:

```
kk apply -f  rollout-deploy-server-ISTIO-ConfigMap.yaml
kk apply -f config-map-01.yaml


```

Po wgraniu nowej wersji rolloutu (to oczywiście ten sam AR ale z jedną małą zmianą bo ten AR używa od tej pory CM) pojawia sie nowy rollout , zaś jak się wejdzie na PODa to widać zmienne z ConfigMapy:
```$ kk exec -ti test-rollout-istio-8675765c8c-4ffmt bash
nginx@test-rollout-istio-8675765c8c-4ffmt:/$ cat /config/key01 ; echo
alamakota
nginx@test-rollout-istio-8675765c8c-4ffmt:/$ cat /config/key02 ; echo
ma
nginx@test-rollout-istio-8675765c8c-4ffmt:/$ cat /config/key03 ; echo
kota
```
podmieńmy teraz ale znowu nie obraz ale ConfigMmapę - załadujmy kolejną nową:
```$ cat config-map-02.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-02
data:
  key01: "alamakota 2"
  key02: "ma 2 "
  key03: "kota 2"

$ kk apply -f  config-map-02.yaml
configmap/cm-02 created

$ kk get cm
NAME               DATA   AGE
cm-01              3      17m
cm-02              3      6s
kube-root-ca.crt   1      5h


```


idąc za: https://argo-rollouts.readthedocs.io/en/stable/getting-started/#2-updating-a-rollout

*Just as with Deployments, any change to the Pod template field (spec.template) results in a new version (i.e. ReplicaSet) to be deployed.* * Updating a Rollout involves modifying the rollout spec, typically changing the container image field with a new version, and then running kubectl apply against the new manifest. As a convenience, the rollouts plugin provides a set image command, which performs these steps against the live rollout object in-place*


widać że jakakolwiek modyfikacja AR via set image czy jakiekolwiek zmiany w definicji AR powodują uruchomienie mechanizmu rolloutu 

Jedno co dziwi to jak widać do set image dorobiono CLI (kubectl argo rollouts set image test-rollout-istio app07=gimboo/nginx_nonroot2) a do innych modyfikacji już nie 

Zatem musimy sami sobie zmienić w rollout.yaml wskazanie na inną config-mapę 

```
$ diff rollout-deploy-server-ISTIO-ConfigMap.yaml rollout-deploy-server-ISTIO-ConfigMap-02.yaml
43c43
<           name: cm-01
---
>           name: cm-02

$ kk apply -f  rollout-deploy-server-ISTIO-ConfigMap-02.yaml
rollout.argoproj.io/test-rollout-istio configured

```

pojawił się nowy POD i pojawił się nowy ROLLOUT 
niby mała zmiana (zamontowanie innej CM) a jednak jest zmianą -  więc AR powołało nową revision i wstrzymało ją z wagą na 5% w VS

```
$ kubectl argo rollouts get rollout test-rollout-istio
Name:            test-rollout-istio
Namespace:       test-ar-istio
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/2
  SetWeight:     5
  ActualWeight:  5
Images:          gimboo/nginx_nonroot (canary, stable)
Replicas:
  Desired:       1
  Current:       2
  Updated:       1
  Ready:         2
  Available:     2

NAME                                            KIND        STATUS        AGE   INFO
⟳ test-rollout-istio                            Rollout     ॥ Paused      5h4m  
├──# revision:12                                                                
│  └──⧉ test-rollout-istio-549bbd66c6           ReplicaSet    Healthy     18s   canary
│     └──□ test-rollout-istio-549bbd66c6-h5km8  Pod           Running     18s   ready:2/2
├──# revision:11                                                                
│  └──⧉ test-rollout-istio-8675765c8c           ReplicaSet    Healthy     21m   stable
│     └──□ test-rollout-istio-8675765c8c-4ffmt  Pod           Running     21m   ready:2/2


$ kk exec -ti test-rollout-istio-8675765c8c-4ffmt -- cat /config/key01 ; echo
alamakota
$ kk exec -ti test-rollout-istio-549bbd66c6-h5km8 -- cat /config/key01 ; echo
alamakota 2

```
warto zaznaczyć że w AR da się podmieniać via CLI tylko image - jakiekolwiek inne zmiany trzeba robić ręcznie modyfikując rollout spec

zaś modyfikując AR otrzymujemy nową rewizję AR 

*Updating a Rollout involves modifying the rollout spec*



### Dodajmy teraz GATEWAY 
wykonuje sie to normalnie , jak w klasyku ISTIO, czyli dodając sekcje gateway do definicji VS:

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtservice-app07
spec:
  hosts:
  - app07.test-ar-istio.svc.cluster.local
  - uk.bookinfo7.com
  gateways:
  - mesh
  - test-ar-istio/mygateway
  http:
  - route:
    - destination:
        host: app07.test-ar-istio.svc.cluster.local
        subset: version-v2
      weight: 50
    - destination:
        host: app07.test-ar-istio.svc.cluster.local
        subset: version-v1
      weight: 50
    name: primary
```

ogólnie można sobie wygodnie porównać zwykłe wdrożenie Traffic-Splittingu via ISTIO-bez-AR z takim gdzie to AR zarządza ISTIO 

w repo w katalogu AR-z-ISTIO-SubsetLevelTrafficSplitting jest podfolder ISTIO-classic a w nim wszystkie obiekty do wdrożenia drugiego zestawu usług - jak sie wdroży obiekty z niego to mamy stan gdzie app07 jest oparte o ArgoRollouts a app08 to gołe ISTIO 

```

$ kk get vs
NAME                GATEWAYS                             HOSTS                                                          AGE
virtservice-app07   ["mesh","test-ar-istio/mygateway"]   ["app07.test-ar-istio.svc.cluster.local","uk.bookinfo7.com"]   7h51m
virtservice-app08   ["mesh","test-ar-istio/mygateway"]   ["app08.test-ar-istio.svc.cluster.local","uk.bookinfo8.com"]   36m
$ kk get dr
NAME             HOST                                    AGE
destrule-app07   app07.test-ar-istio.svc.cluster.local   7h52m
destrule-app08   app08.test-ar-istio.svc.cluster.local   37m
$ kk get gateway
NAME        AGE
mygateway   66m
```


(uwaga, gateway występuje 2 razy, w 2 plikach - są niemal identyczne ale ten drugi uzupełnia GW o dodatkowe hosty dla app08 (wystarczy zrobić diff na tych 2 plikach) )
```$ diff EXPOSE_gateway_simple_MATCH_URI_PREFIX.yaml ISTIO-classic/EXPOSE_gateway_simple_MATCH_URI_PREFIX.yaml
18,19c18,19
<     #- app08.test-ar-istio.svc.cluster.local
<     #- uk.bookinfo8.com
---
>     - app08.test-ar-istio.svc.cluster.local
>     - uk.bookinfo8.com
```
### TESTY :
testy na k8s-svc (wewnątrz klastra) wykonujemy z PODa consumera a testy połączenia via GW z jakiejś stacji na zewnątrz klastra (np z GCE stojącej obok węzłów GKE) 

```

nginx@consumer-58f6fd4c95-gnrmt:/$ curl app07:8080
test-rollout-istio-dd4cc6cb4-lvjw2
<br>2
nginx@consumer-58f6fd4c95-gnrmt:/$ curl app07:8080
test-rollout-istio-dd4cc6cb4-lvjw2
<br>2

$ kk get svc -n istio-system
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.108.1.52    10.128.0.5    15020:32604/TCP,443:30467/TCP,80:30423/TCP   5h45m
istiod                 ClusterIP      10.108.5.9     <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        5h45m
istiod-asm-1162-2      ClusterIP      10.108.13.34   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        5h45m

slawek@instance-1:~$ curl -k --resolve uk.bookinfo7.com:80:10.128.0.5 http://uk.bookinfo7.com
test-rollout-istio-dd4cc6cb4-lvjw2
<br>2
slawek@instance-1:~$

```


## Host-level traffic splitting




wróćmy do metody omawianej w getting-started:
https://argo-rollouts.readthedocs.io/en/stable/getting-started/istio/

omawiają tu bowiem nieco inną koncepcję - Host-level Traffic Splitting
*The first approach to traffic splitting using Argo Rollouts and Istio, is splitting between two hostnames, or Kubernetes Services: a canary Service and a stable Service*


Folder w tym repo z plikami YAML: AR-z-ISTIO-HostLevel-LevelTrafficSplitting

```
kubectl create ns test-ar-istio-2
kubectl config set-context --current --namespace=test-ar-istio-2
kubectl -n istio-system get pods -l app=istiod --show-labels | grep rev
kubectl label namespace test-ar-istio-2 istio-injection- istio.io/rev=asm-1162-2 --overwrite```
Koncepcja ta bazuje na AR który definiuje :
```
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: test-rollout-istio
spec:
  replicas: 1
  strategy:
    canary:
      canaryService: canary-service
      stableService: stable-service
      trafficRouting:
        istio:
          virtualServices:
          - name: virtservice-app07
            routes:
            - primary          
      steps:
      - setWeight: 5
      - pause: {}
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      name: app07
  template:
    metadata:
      labels:
        name: app07
    spec:
      containers:
      - image: gimboo/nginx_nonroot
        name: app07
        resources: {}
```
w definicji AR jak widać jest wskazanie na dwa k8s-svc (canary-service i stable-service) oraz na virt-service 
w wyniku działania AR powstaje 1 POD (lub 2 PODy jesli następuje wdrożenie nowej wersji rolloutu) 
PODy te mają dodawane rollouts-pod-template-hash
jednocześnie AR modyfikuje nasze k8s-SVC tak że ich selectory poza oryginalnym selector-name-app07 mają również 
```
  selector:
    name: app07
    rollouts-pod-template-hash: dd4cc6cb4
```
w chwili gdy jest tylko jedna rewizja rolloutu te SVC wskazują na to samo:
```
$ kk get po -o wide --show-labels
NAME                                 READY   STATUS    RESTARTS   AGE   IP            NODE                                     NOMINATED NODE   READINESS GATES   LABELS
test-rollout-istio-dd4cc6cb4-bjzkv   2/2     Running   0          22m   10.104.0.23   gke-central-default-pool-6ac74c87-ldjn   <none>           <none>            name=app07,rollouts-pod-template-hash=dd4cc6cb4,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=test-rollout-istio-dd4cc6cb4,service.istio.io/canonical-revision=latest,topology.istio.io/network=a-r-008-default


$ kk get svc
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
canary-service   ClusterIP   10.108.8.99    <none>        8080/TCP   43m
stable-service   ClusterIP   10.108.6.132   <none>        8080/TCP   43m

$ kk get svc stable-service -o yaml
[...]  selector:
    name: app07
    rollouts-pod-template-hash: dd4cc6cb4
[...]
$ kk get svc canary-service -o yaml
[...]  selector:
    name: app07
    rollouts-pod-template-hash: dd4cc6cb4
[...]
$ kk get ep
NAME             ENDPOINTS          AGE
canary-service   10.104.0.23:8080   43m
stable-service   10.104.0.23:8080   43m
```

z kolei serce układu czyli VirtService ma 2 destination , każda na inny k8s-svc , na stable-service ma w=100, na canary-service w=0 

```
$ kk get vs
NAME                GATEWAYS   HOSTS                                                            AGE
virtservice-app07              ["app07-vs","uk.bookinfo7.com"]   42m

$ kk get vs virtservice-app07 -o yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: virtservice-app07
  namespace: test-ar-istio-2
spec:
  hosts:
  - app07-vs
  - uk.bookinfo7.com
  http:
  - name: primary
    route:
    - destination:
        host: stable-service
      weight: 100
    - destination:
        host: canary-service
      weight: 0

```

zaraz po  wykonaniu modyfikacji :

```
$ kubectl argo rollouts set image test-rollout-istio app07=gimboo/nginx_nonroot3
rollout "test-rollout-istio" image updated
```

pojawia się drugi POD z innym rollouts-pod-template-hash :

```
$ kk get po -o wide --show-labels
NAME                                 READY   STATUS    RESTARTS   AGE   IP            NODE                                     NOMINATED NODE   READINESS GATES   LABELS
test-rollout-istio-5bfbf9599-xl7mv   2/2     Running   0          10s   10.104.1.23   gke-central-default-pool-6ac74c87-w6d2   <none>           <none>            name=app07,rollouts-pod-template-hash=5bfbf9599,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=test-rollout-istio-5bfbf9599,service.istio.io/canonical-revision=latest,topology.istio.io/network=a-r-008-default
test-rollout-istio-dd4cc6cb4-bjzkv   2/2     Running   0          26m   10.104.0.23   gke-central-default-pool-6ac74c87-ldjn   <none>           <none>            name=app07,rollouts-pod-template-hash=dd4cc6cb4,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=test-rollout-istio-dd4cc6cb4,service.istio.io/canonical-revision=latest,topology.istio.io/network=a-r-008-default

```
k8s-SVC już wyglądają inaczej (tzn stable-service wygląda tak samo, ale canary-service zmienił sie tak że jego selector wyłapuje nowego PODa) :

```
$ kk get svc stable-service -o yaml | grep rollouts-pod-template-hash
    rollouts-pod-template-hash: dd4cc6cb4
$ kk get svc canary-service -o yaml | grep rollouts-pod-template-hash
    rollouts-pod-template-hash: 5bfbf9599

$ kk get ep
NAME             ENDPOINTS          AGE
canary-service   10.104.1.23:8080   49m
stable-service   10.104.0.23:8080   49m

```

Virt-service zmienił wagi:
```
$ kk get vs virtservice-app07 -o yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: virtservice-app07
  namespace: test-ar-istio-2
spec:
  hosts:
  - app07-vs
  - uk.bookinfo7.com
  http:
  - name: primary
    route:
    - destination:
        host: stable-service
      weight: 95
    - destination:
        host: canary-service
      weight: 5

```


