# Ingress-класс ALB

Данное руководство описывает, как развернуть два Ingress-ресурса в одном кластере с использованием Ingress-контроллеров NGINX и ALB, а также маршрутизировать через них трафик к одному и тому же приложению.

## Предварительные условия

- Установленная и настроенная утилита `yc`.
- Публичная зона для [делегированного](https://yandex.cloud/docs/dns/operations/zone-create-public?from=int-console-help-center-or-nav) домена.
- Развернутый mK8S в YC.
- Заполненные переменные в файлах `app/alb-ing.yaml` и `app/nginx-ing.yaml`.

```bash
git clone https://github.com/yc-architect-solution-library
cd yc-architect-solution-library/yc-k8s-ingress-class
```

## Установка NGINX ingress и cert-manager.io

```bash
kubectl create ns nginx
kubectl create ns app

# NGINX Ingress
kubectl config set-context --current --namespace nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx

kubectl get svc -n nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.96.145.247   84.201.163.76   80:31668/TCP,443:30353/TCP   6d12h # copy External IP of nginx-ingress-nginx-controller service
ingress-nginx-controller-admission   ClusterIP      10.96.200.103   <none>          443/TCP                      6d12h




yc dns zone add-records your-zone  --record  "*.nginx.<DOMAIN>. 60 A <EXT_IP>"
yc dns zone list-records your-zone

+-------------------------+------+------+--------------------------------+
|          NAME           | TTL  | TYPE |              DATA              |
+-------------------------+------+------+--------------------------------+
| *.nginx.example.com.    |   60 | A    | <EXT_IP>                       |
| example.com.            | 3600 | NS   | ns1.yandexcloud.net.           |
|                         |      |      | ns2.yandexcloud.net.           |
| example.com.            | 3600 | SOA  | ns1.yandexcloud.net.           |
|                         |      |      | mx.cloud.yandex.net. 1 10800   |
|                         |      |      | 900 604800 900                 |
+-------------------------+------+------+--------------------------------+


### DNS Challenge Webhook

- Install [Cert-manager with CloudDNS ACME webhookubectl plugin from Marketplace](https://cloud.yandex.ru/marketplace/products/yc/cert-manager-webhook-yandex)


```bash
kubectl config set-context --current --namespace app
kubectl apply -f app/demo-app1.yaml && kubectl apply -f app/demo-app2.yaml
kubectl apply -f app/nginx-ing.yaml
```

```bash
# Проверка корректной работы всех компонентов
curl https://app.nginx.<DOMAIN>
curl https://app.nginx.<DOMAIN>/app1
curl https://app.nginx.<DOMAIN>/app2
App by Ingress Class
```

### ALB Ingress Class Installation

- Install [ALB Ingress Controller](https://cloud.yandex.ru/marketplace/products/yc/alb-ingress-controller) from Marketplace (follow the instructions)

Then do:

```bash
kubectl config set-context --current --namespace alb
kubectl apply -f alb-ingress-class.yaml # apply ALB ingressClass definition

kubectl config set-context --current --namespace app
kubectl apply -f app/alb-ing.yaml

kubectl get svc # copy External IP of alb-ingress service
yc dns zone add-records your-zone  --record  "*.alb.<DOMAIN>. 60 A <EXT_IP>"
yc dns zone list-records your-zone

+-------------------------+------+------+--------------------------------+
|          NAME           | TTL  | TYPE |              DATA              |
+-------------------------+------+------+--------------------------------+
| *.alb.example.com.      |   60 | A    | <EXT_IP>                       |
| *.nginx.example.com.    |   60 | A    | <EXT_IP>                       |
| example.com.            | 3600 | NS   | ns1.yandexcloud.net.           |
|                         |      |      | ns2.yandexcloud.net.           |
| example.com.            | 3600 | SOA  | ns1.yandexcloud.net.           |
|                         |      |      | mx.cloud.yandex.net. 1 10800   |
|                         |      |      | 900 604800 900                 |
+-------------------------+------+------+--------------------------------+

# Проверка корректной работы всех компонентов
curl https://app.alb.<DOMAIN>
curl https://app.alb.<DOMAIN>/app1
curl https://app.alb.<DOMAIN>/app2
```
