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

nastepnie labelujemy ten namespaces zeby działał autoinject dla envoy-proxies:
```
kubectl label namespace test-istio istio-injection=enabled --overwrite
kubectl label namespace test-istio istio-injection- istio.io/rev=asm-182-2 --overwrite
kubectl label namespace test-istio istio-injection- istio.io/rev=asm-1162-2 --overwrite
```






# ARGO ROLLOUTS BASICS




## dwa
### trzy

text
text
