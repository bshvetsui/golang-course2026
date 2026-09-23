**Flannel** — простой CNI: его основная задача — дать Pod'ам на разных Node сетевую связность. Это самый минималистичный из трёх вариантов.

`Pod → veth → Flannel → VXLAN/route → другая Node → Pod`

**Возраст:** ~12 лет. **Позиционирование:** старый и простой CNI, не мёртвый, но по возможностям значительно беднее Calico/Cilium.

## 1. Что умеет и чего не умеет

1. **Pod networking — да.** Flannel создаёт сеть Pod'ов и маршруты между Node. Это его главная задача.
2. **VXLAN — да.** Можно завернуть Pod traffic в UDP между Node. Удобно, если физическая сеть ничего не знает о Pod CIDR.
3. **Native routing / host-gw — да.** Можно отказаться от VXLAN и использовать обычные Linux routes, если Node имеют подходящую L2/L3 связность.
4. **NetworkPolicy — базово через отдельный controller.** Сам `flanneld` policy не реализует, но современный Flannel умеет устанавливать `kube-network-policies`. Это стандартные Kubernetes NetworkPolicy, без богатых Calico/Cilium extensions.
5. **BGP — нет.** Flannel сам не занимается распространением Pod routes через BGP. У Calico это одна из главных функций, у Cilium BGP тоже есть.
6. **eBPF dataplane — нет.** Основная архитектура Flannel не построена на eBPF. У Cilium eBPF — фундамент, у Calico есть eBPF dataplane как вариант.
7. **kube-proxy replacement — нет.** Flannel занимается Pod network, а Services продолжает обслуживать `kube-proxy`. Cilium может заменить kube-proxy полностью.
8. **L7 Policy — нет.** Нельзя написать правило уровня `разрешить GET /api, запретить POST /admin`. Cilium это умеет через L7 policy.
9. **Observability — минимальная.** Есть обычные логи/метрики, но нет встроенной системы анализа network flows уровня Hubble.
10. **UI — встроенного UI нет.** Диагностика в основном через `kubectl`, `ip route`, `tcpdump`, логи.
11. **Gateway API / Ingress implementation — нет.** Для ingress/gateway нужен отдельный controller вроде NGINX/Envoy/Traefik.
12. **Service Mesh — нет.** Flannel не занимается mTLS, L7 routing, traffic splitting и service-to-service policy.
13. **High-scale networking — не основная цель.** Flannel хорош именно своей простотой; если нужен большой набор networking/security/observability возможностей, обычно смотрят Calico/Cilium.

## 2. Архитектура

`kubelet → CNI plugin → flanneld → Linux networking`

Пример адресации:

```text
Cluster Pod CIDR: 10.244.0.0/16
node-1: 10.244.1.0/24
node-2: 10.244.2.0/24
node-3: 10.244.3.0/24
```

На каждой Node работает `kube-flannel` DaemonSet.

## 3. Установка kubeadm

```bash
# Pod CIDR должен совпадать с сетью Flannel
kubeadm init --pod-network-cidr=10.244.0.0/16

# Простая установка Flannel
kubectl apply -f \
https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Через Helm:

```bash
# Добавить repository
helm repo add flannel https://flannel-io.github.io/flannel/

# Namespace
kubectl create namespace kube-flannel

# Flannel требует privileged networking
kubectl label --overwrite ns kube-flannel \
pod-security.kubernetes.io/enforce=privileged

# Установка
helm install flannel flannel/flannel \
--namespace kube-flannel \
--set podCidr="10.244.0.0/16"
```

## 4. VXLAN

Самый обычный режим:

`Pod → Node → flannel.1 → VXLAN UDP/8472 → Node → Pod`

Зачем нужен VXLAN: физическим router'ам не требуется знать маршруты `10.244.x.x`.

Проверить:

```bash
# VXLAN interface
ip addr show flannel.1

# Маршруты Pod CIDR
ip route

# Посмотреть VXLAN traffic
sudo tcpdump -ni any udp port 8472
```

Важно разрешить между Node:

```text
UDP 8472
```

Минус VXLAN — encapsulation добавляет overhead, поэтому нужно следить за MTU.

## 5. host-gw

Без overlay:

`Pod → Linux route → remote Node → Pod`

Конфигурация:

```json
{
  "Network": "10.244.0.0/16",
  "Backend": {
    "Type": "host-gw"
  }
}
```

Зачем: меньше overhead, чем VXLAN.

Ограничение: Node должны нормально маршрутизироваться между собой; для сложного DC routing Calico+BGP обычно удобнее.

## 6. NetworkPolicy

Включить controller:

```bash
# Flannel + Kubernetes NetworkPolicy controller
helm install flannel flannel/flannel \
--namespace kube-flannel \
--set podCidr="10.244.0.0/16" \
--set netpol.enabled=true
```

После этого можно применять стандартные Kubernetes NetworkPolicy.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

Смысл:

```text
frontend → backend:8080 ✅
другие Pods → backend ❌
```

Но здесь нет Calico `GlobalNetworkPolicy`, policy tiers и богатых selectors и нет Cilium L7/FQDN/HTTP policy.

## 7. Настройка

Посмотреть ConfigMap:

```bash
# Конфигурация Flannel
kubectl -n kube-flannel get cm kube-flannel-cfg -o yaml
```

Типично настраиваются: `Pod CIDR`, `VXLAN/host-gw`, `MTU`, `IPv4/IPv6`, masquerading.

## 8. Проверка

```bash
# Flannel должен быть на каждой Node
kubectl -n kube-flannel get pods -o wide

# DaemonSet
kubectl -n kube-flannel get ds

# Pod IP
kubectl get pods -A -o wide

# PodCIDR каждой Node
kubectl get nodes \
-o custom-columns=NODE:.metadata.name,PODCIDR:.spec.podCIDR

# Routes
ip route

# Flannel runtime config
cat /run/flannel/subnet.env
```

## 9. Debug

Если Pod на разных Node не общаются:

1. `kubectl -n kube-flannel get pods` — жив ли Flannel.
2. `kubectl get nodes ...` — есть ли PodCIDR.
3. `ip route` — появились ли маршруты.
4. `ip addr show flannel.1` — существует ли VXLAN interface.
5. Проверить `UDP/8472` между Node.
6. Проверить MTU.
7. Проверить `/etc/cni/net.d/`.
8. Проверить `/opt/cni/bin/`.
9. Проверить `br_netfilter`.

```bash
# Проверить kernel module
lsmod | grep br_netfilter

# Загрузить при необходимости
sudo modprobe br_netfilter

# Логи Flannel
kubectl -n kube-flannel logs \
-l app=flannel -c kube-flannel
```

## 10. Когда выбирать

1. Lab/homelab.
2. Маленький простой Kubernetes.
3. Нужна только нормальная Pod-to-Pod сеть.
4. Не нужны BGP, Hubble, L7 policy, встроенный Gateway API и Service Mesh.

**Коротко:** `Flannel = минимальный CNI → Pod networking → обычно VXLAN → простой в эксплуатации → мало дополнительных возможностей`.
