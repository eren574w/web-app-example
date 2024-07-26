# Web App Example with Stress Testing

Bu proje, Kubernetes üzerinde stress testing yapabilmek için oluşturulmuştur. Minikube kullanarak bir yerel Kubernetes kümesi kuracak ve bir uygulamanın CPU ve bellek kullanımını test etmek için Horizontal Pod Autoscaler (HPA) yapılandırmasını gerçekleştireceğiz.

## İçindekiler

- [Gereksinimler](#gereksinimler)
- [Kurulum](#kurulum)
  - [Minikube Kurulumu](#minikube-kurulumu)
  - [Metrics Server Kurulumu](#metrics-server-kurulumu)
- [Yapılandırmalar](#yapılandırmalar)
  - [Stress Deployment](#stress-deployment)
  - [HPA Yapılandırması](#hpa-yapılandırması)
- [Test Etme](#test-etme)
- [Temizlik](#temizlik)

## Gereksinimler

- **Minikube**: Kubernetes kümesi oluşturmak için.
- **kubectl**: Kubernetes komut satırı aracı.
- **kubectl-plugins** (isteğe bağlı): Minikube komutlarını yönetmek için.

### Minikube ve kubectl Kurulumu

```bash
# Minikube kurulum komutları (Ubuntu için)
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube && sudo mv minikube /usr/local/bin/

# Kubectl kurulum komutları
curl -LO "https://dl.k8s.io/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

# Minikube başlatma
minikube start




# Metrics Server'ı etkinleştirin
minikube addons enable metrics-server


# dosyaları oluşturma

touch stress-deployment.yaml stress-hpa.yaml

# stress-deployment.yaml dosyası : 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stress-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stress
  template:
    metadata:
      labels:
        app: stress
    spec:
      containers:
      - name: stress
        image: alpine
        command: ["/bin/sh", "-c", "apk add --no-cache stress-ng && stress-ng --cpu 2 --timeout 60s"]
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"




# HPA dosyası: 

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: stress-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: stress-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 50


# deployment ve hpa yı çalıştıralım
kubectl apply -f stress-deployment.yaml
kubectl apply -f stress-hpa.yaml

# kontrol edelim 
kubectl get pods
kubectl get hpa


# 
kubectl rollout restart deployment metrics-server -n kube-system

# pod ismini alın 
kubectl get pod

# gpu ve memory i çalıştıralım 
kubectl exec -it <pod_name> -- /bin/sh

stress-ng --cpu 5 --timeout 60s


# test edelim 
kubectl get pod
kubectl get hpa




# temizlik
kubectl delete -f stress-deployment.yaml
kubectl delete -f stress-hpa.yaml
minikube stop
minikube delete

