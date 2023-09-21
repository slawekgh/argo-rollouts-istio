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

Powyższe to podstawy istiowych podstaw ale trzeba to mieć przed oczami przy analizie modeli opertych na Argo-Rollout




# ARGO ROLLOUTS BASICS




## dwa
### trzy

text
text
