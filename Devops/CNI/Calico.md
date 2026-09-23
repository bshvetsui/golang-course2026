**Calico** — production CNI, ориентированный не только на Pod networking, но и на **routing + BGP + NetworkPolicy**.

`Pod → Calico → Linux routing/VXLAN/BGP → Node/router → Pod`

**Возраст:** ~12 лет. **Позиционирование:** зрелый production standard; особенно логичен для on-prem/bare metal и инфраструктуры, где DevOps управляет routing/BGP.

## 1. Что умеет и зачем

1. **Pod networking — да.** Полноценный CNI с собственным IPAM.
2. **VXLAN — да.** Можно построить overlay как во Flannel, если физическая сеть не должна знать Pod CIDR.
3. **Native routing — да.** Можно передавать Pod traffic обычной маршрутизацией без encapsulation.
4. **BGP — одна из главных фич.** Calico Node может сообщать router'ам: `10.244.1.0/24 находится через node-1`. Это позволяет нормально интегрировать Kubernetes в физическую сеть DC. Flannel этого сам не делает; Cilium BGP поддерживает, но исторически Calico особенно силён здесь.
5. **NetworkPolicy — очень сильная сторона.** Есть стандартные Kubernetes policy + собственные Calico Policy.
6. **GlobalNetworkPolicy — да.** Можно задать security policy сразу на весь cluster, а не копировать её по Namespace.
7. **Policy tiers/order — да.** Можно определить, что security baseline должен обрабатываться раньше application policy.
8. **eBPF — есть.** Calico умеет eBPF dataplane, но в отличие от Cilium проект не является исключительно eBPF-first.
9. **kube-proxy replacement — возможен в eBPF dataplane.** Но это не настолько центральная часть идентичности Calico, как у Cilium.
10. **L7 Policy — ограниченнее Cilium/зависит от дополнительных компонентов и редакции.** Основная сила open-source Calico — L3/L4 network security.
11. **Observability — есть.** Современный Calico Open Source имеет flow logs через `Goldmane`.
12. **UI — есть Calico Whisker.** Это web UI для просмотра network flows, source/destination, policy verdict и фильтрации по Pod/Namespace/Policy. В Calico OSS 3.32 функция пока помечена Tech Preview.
13. **Enterprise UI — значительно богаче.** Calico Enterprise Manager имеет Dashboard, Service Graph, policy, alerts, packet capture и другие функции.
    [[Devops/images/Pasted image 20260923101114.png|Скриншот Calico Enterprise Manager]]
14. **Gateway API / Service Mesh — не главная причина выбирать Calico.** Для этих задач Cilium сейчас предоставляет более цельный встроенный стек.
15. **High-scale networking — да.** BGP, route reflectors, IPAM blocks и отсутствие обязательного overlay делают Calico подходящим для больших on-prem сетей.

## 2. Архитектура

`kubelet → Calico CNI → IPAM → calico-node/Felix → Linux dataplane`

Главный агент — **Felix**. Он настраивает routes, policy, NAT, interfaces и выбранный dataplane.

При BGP:

`Calico Node → BGP → другие Node / Route Reflector / ToR Router`

## 3. Установка

Современный вариант — Tigera Operator:

```bash
# Добавить официальный repository
helm repo add projectcalico https://docs.tigera.io/calico/charts
helm repo update

# Namespace operator
kubectl create namespace tigera-operator

# Установить operator
helm install calico projectcalico/tigera-operator \
--namespace tigera-operator
```

Проверить:

```bash
# Основные Calico Pods
kubectl get pods -n calico-system

# Operator
kubectl get pods -n tigera-operator
```

## 4. VXLAN

`Pod → Calico → VXLAN → remote Node → Pod`

Пример IPPool:

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: pods
spec:
  cidr: 10.244.0.0/16
  vxlanMode: CrossSubnet
  natOutgoing: true
```

`CrossSubnet` означает: если Node находятся в одном subnet, можно идти напрямую; если в разных — использовать VXLAN.

Зачем: меньше encapsulation, чем при VXLAN абсолютно между всеми Node.

## 5. BGP

Пример:

```text
node-1 PodCIDR: 10.244.1.0/24
node-2 PodCIDR: 10.244.2.0/24

node-1 → BGP advertise 10.244.1.0/24
node-2 → BGP advertise 10.244.2.0/24
```

Подключение внешнего router:

```yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: tor-router
spec:
  peerIP: 10.0.0.1
  asNumber: 65001
```

Получается:

`Pod → Node → обычный IP route → ToR/router → другая Node → Pod`

Зачем это важно: Kubernetes network становится частью обычной DC network, а не отдельным VXLAN overlay.

Для большого количества Node вместо full-mesh обычно используют **Route Reflector**:

`Nodes → Route Reflector → routes → Nodes`

## 6. NetworkPolicy

Обычная Kubernetes policy:

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

`frontend → backend:8080 ✅ | другие → backend ❌`

## 7. Calico NetworkPolicy

Собственный CRD богаче Kubernetes NetworkPolicy:

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: backend
  namespace: production
spec:
  selector: app == 'backend'
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == 'frontend'
      destination:
        ports:
          - 8080
```

Зачем собственный format: больше selectors, explicit `Allow/Deny`, policy ordering и другие Calico возможности.

## 8. GlobalNetworkPolicy

Работает cluster-wide:

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: database-security
spec:
  order: 10
  selector: app == 'database'
  ingress:
    - action: Allow
      source:
        selector: app == 'backend'
```

Это удобно для правил уровня платформы:

```text
1. security team policy
2. platform policy
3. application policy
```

То есть приложение не должно самостоятельно повторять одинаковый baseline в 50 Namespace.

## 9. IPAM

Calico распределяет Pod IP по блокам:

`IPPool → blocks → Node → Pods`

Проверить:

```bash
# Все IP pools
calicoctl get ippool -o wide

# Calico Nodes
calicoctl get nodes -o wide
```

Зачем blocks: вместо тысячи отдельных Pod routes можно агрегировать маршруты и уменьшить routing table.

## 10. eBPF dataplane

Классический Calico:

`Packet → Linux routing + iptables/nftables → destination`

eBPF mode:

`Packet → Calico eBPF programs/maps → destination`

Зачем eBPF: уменьшить зависимость от больших iptables rulesets, ускорить Service handling и получить более программируемый dataplane.

Но если задача звучит именно как **eBPF-first Kubernetes networking**, чаще первым рассматривают Cilium.

## 11. Observability и UI

Open Source Calico имеет:

`Felix flow information → Goldmane → Whisker Web UI`

Whisker показывает real-time flow logs и позволяет фильтровать их по Pod, Namespace, Policy и другим параметрам. В текущем Calico OSS это Tech Preview.

Открыть UI:

```bash
# Локальный доступ к Whisker
kubectl port-forward \
-n calico-system service/whisker 8081:8081

# Затем открыть localhost:8081
```

Если Whisker не включён:

```bash
# Flow log API
kubectl apply -f - <<'EOF'
apiVersion: operator.tigera.io/v1
kind: Goldmane
metadata:
  name: default
EOF

# Web UI
kubectl apply -f - <<'EOF'
apiVersion: operator.tigera.io/v1
kind: Whisker
metadata:
  name: default
EOF
```

Enterprise версия идёт дальше: Service Graph, dashboards, alerts, packet capture и Kibana.

## 12. Проверка

```bash
# Calico Pods
kubectl get pods -n calico-system -o wide

# IP pools
calicoctl get ippool -o wide

# BGP sessions
calicoctl node status

# Policies
calicoctl get networkpolicy -A

# Global policies
calicoctl get globalnetworkpolicy

# Linux routes
ip route

# Calico interfaces
ip link | grep cali
```

## 13. Debug

1. Pod↔Pod не работает → `ip route`, `cali*` interfaces, IPPool, MTU.
2. VXLAN не работает → проверить tunnel interface/firewall.
3. BGP не работает → `calicoctl node status`, `TCP/179`, ASN, BGPPeer.
4. Traffic неожиданно DROP → labels, selectors, Namespace policy, GlobalNetworkPolicy.
5. Непонятно кто кого блокирует → смотреть flow logs через Whisker.

## 14. Когда выбирать

1. On-prem/bare metal.
2. Нужен BGP с физическими router'ами.
3. Нужны сильные L3/L4 NetworkPolicy.
4. Нужны GlobalNetworkPolicy и централизованные security rules.
5. Есть существующая инфраструктура Calico.
6. Не обязательно нужен Cilium-стек из Hubble + Gateway API + Service Mesh.

**Коротко:** `Calico = CNI + IPAM + routing + BGP + сильные NetworkPolicy + хороший вариант для серьёзного on-prem networking`.
