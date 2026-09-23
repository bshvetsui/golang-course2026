**Cilium** — современный eBPF-first CNI. Он старается закрыть не только Pod networking, но и Service load balancing, policy, observability, ingress/gateway и часть service mesh задач.

`Pod/Application → eBPF → Linux kernel → Service/Pod/Node`

**Возраст:** ~10 лет. **Позиционирование:** самый modern/eBPF-oriented из Flannel, Calico и Cilium.

## 1. Что умеет и почему это важно
1. **eBPF dataplane — основа Cilium.** Вместо того чтобы описывать всю логику тысячами iptables rules, Cilium загружает программы в Linux kernel и хранит состояние в BPF Maps. Это делает dataplane более программируемым и позволяет Cilium объединить networking, policy и observability.
2. **kube-proxy replacement — да.** Cilium может сам реализовать `ClusterIP`, `NodePort`, `LoadBalancer` и Service backend selection через eBPF. Flannel этого не делает; Calico имеет eBPF возможности, но Cilium особенно заточен под этот сценарий.
3. **NetworkPolicy — да.** Поддерживает стандартную Kubernetes NetworkPolicy и собственные более мощные policy.
4. **L7 Policy — да.** Можно фильтровать не только `IP:port`, но и HTTP method/path, DNS/FQDN, gRPC и т.п. Например `GET /public` разрешить, а `POST /admin` запретить.
5. **Observability — встроена.** Cilium знает не только что пакет был DROP, но может показать source workload, destination workload, policy verdict, DNS/HTTP context и причину DROP.
6. **Hubble — observability система поверх Cilium/eBPF.** Она собирает network flows со всех Cilium agents и позволяет смотреть связь сервисов.
7. **Hubble UI — полноценный web UI.** Показывает Service Map и network flows между workloads. То есть при проблеме можно визуально увидеть `frontend → backend → database`, а не только читать `tcpdump`.
    [[Devops/images/Pasted image 20260923101207.png|Схема Hubble UI]]
8. **Gateway API — да.** Cilium может выступать dataplane для Kubernetes Gateway API: ingress, HTTP routing, traffic splitting и другие north-south/L7 задачи. Это уменьшает потребность в отдельном NGINX Ingress/Envoy Gateway стеке.
9. **Service Mesh — да.** Может давать service-to-service security, traffic management, encryption и observability. Часть возможностей реализуется через eBPF, а L7 через Envoy. Это позволяет не ставить sidecar proxy в каждый Pod для всех задач.
10. **BGP — да.** Cilium может объявлять Kubernetes routes/services во внешнюю сеть. У Calico BGP исторически более центральная фича, но Cilium также умеет интеграцию с DC routing.
11. **VXLAN/Geneve — да.** Можно использовать обычный overlay.
12. **Native routing — да.** Можно вообще отказаться от overlay и маршрутизировать Pod CIDR обычной сетью.
13. **Encryption — да.** Есть WireGuard/IPsec для защиты traffic между Node/workloads.
14. **High-scale networking — одна из целей архитектуры.** eBPF Maps и kernel hooks позволяют уйти от огромных iptables chains и объединить Service LB, policy и visibility в одном dataplane.
15. **UI/observability заметно сильнее Flannel.** У Flannel встроенного network UI фактически нет. Calico получил Whisker, но в OSS он пока Tech Preview; Hubble является зрелой центральной частью Cilium observability.

## 2. Архитектура

На каждой Node:

`cilium-agent → eBPF programs → BPF Maps → Linux kernel`

Дополнительно:

`Cilium Operator → cluster-level operations`

`Hubble Server → Hubble Relay → Hubble CLI/UI`

`Envoy → L7/Gateway API/Service Mesh функции`

## 3. Установка

Для обычного варианта с существующим kube-proxy:

```bash
# Добавить Helm repository
helm repo add cilium https://helm.cilium.io/
helm repo update

# Установить Cilium
helm install cilium cilium/cilium \
--namespace kube-system
```

Проверить:

```bash
# Общий Cilium status
cilium status

# Cilium agents
kubectl -n kube-system get pods \
-l k8s-app=cilium
```

## 4. kube-proxy replacement

При создании kubeadm cluster:

```bash
# Не устанавливать kube-proxy
kubeadm init \
--skip-phases=addon/kube-proxy
```

Cilium:

```yaml
kubeProxyReplacement: true
```

Классическая схема:

`Service → kube-proxy rules → iptables/nftables → backend Pod`

Cilium:

`Service → eBPF lookup → BPF Map → backend Pod`

Зачем: Service networking обслуживается тем же dataplane, который уже занимается CNI, policy и observability.

Посмотреть Service entries:

```bash
# Services, которые видит Cilium
kubectl -n kube-system exec ds/cilium -- \
cilium-dbg service list

# BPF load-balancer map
kubectl -n kube-system exec ds/cilium -- \
cilium-dbg bpf lb list
```

## 5. BPF Maps

Условно:

```text
Service 10.96.10.5:80
→ BPF Map
→ Pod1 10.244.1.10:8080
→ Pod2 10.244.2.20:8080
→ Pod3 10.244.3.30:8080
```

При появлении Pod Cilium может обновить map вместо построения огромной последовательности iptables rules.

Проверить:

```bash
# BPF maps
kubectl -n kube-system exec ds/cilium -- \
cilium-dbg bpf map list
```

## 6. VXLAN

`Pod → Cilium → VXLAN UDP/8472 → Node → Pod`

```yaml
routingMode: tunnel
tunnelProtocol: vxlan
```

Зачем: простая установка, физической сети не нужны Pod routes.

Минус: encapsulation и MTU overhead.

## 7. Native Routing

`Pod → eBPF/Linux routing → physical network → Node → Pod`

```yaml
routingMode: native
ipv4NativeRoutingCIDR: 10.244.0.0/16
autoDirectNodeRoutes: true
```

Зачем: убрать overlay overhead.

Требование: физическая сеть должна уметь доставлять traffic к Pod CIDR; для сложной topology можно использовать BGP.

## 8. NetworkPolicy

Стандартная Kubernetes NetworkPolicy работает как обычно.

Дополнительно есть:

```text
CiliumNetworkPolicy
CiliumClusterwideNetworkPolicy
```

`CiliumNetworkPolicy` — Namespace scope.

`CiliumClusterwideNetworkPolicy` — весь cluster.

## 9. L3/L4 Policy

Разрешить:

`frontend → backend:8080`

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: backend
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
```

Здесь решение принимается по workload identity/labels, а не только по IP.

## 10. L7 Policy

Например:

```text
frontend → GET /public ✅
frontend → POST /admin ❌
```

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: backend-http
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: "/public$"
```

Это уже уровень **application protocol**, а не обычного firewall. Cilium также умеет policy по DNS/FQDN и другим L7 данным.

## 11. Hubble

Hubble получает события прямо из Cilium/eBPF dataplane:

`eBPF flow events → Cilium agent → Hubble → Relay → CLI/UI`

Включить:

```bash
# Включить Hubble + Relay
cilium hubble enable

# Включить Hubble вместе с UI
cilium hubble enable --ui
```

Hubble Relay агрегирует информацию со всех Node, поэтому можно смотреть network flows cluster-wide.

## 12. Hubble CLI

```bash
# Проверить Hubble
hubble status

# Все live flows
hubble observe

# Только DROP
hubble observe --verdict DROPPED

# Только policy decisions
hubble observe --type policy-verdict
```

Практический смысл: вместо «ping не работает, сейчас буду tcpdump'ить пять Node» можно сразу увидеть:

```text
frontend → backend:8080 → FORWARDED
frontend → database:5432 → DROPPED
Reason: Policy denied
```

## 13. Hubble UI

Запустить:

```bash
# Port-forward + открыть Hubble UI в браузере
cilium hubble ui
```

По умолчанию CLI поднимает локальный port-forward к Hubble UI.

В UI можно видеть Service Map:

```text
frontend
   ↓ HTTP
backend
   ↓ TCP/5432
postgres
```

И анализировать:

1. Кто с кем общается.
2. Какие flows разрешены/запрещены.
3. Source/Destination Namespace и Pod.
4. Network protocol.
5. Policy verdict.
6. L7 visibility при соответствующей конфигурации.

Это очень полезно именно при troubleshooting microservices.

## 14. Gateway API
Gateway API — современный Kubernetes API для управления входящим L4/L7 traffic.

Условно:
`Internet → Gateway → HTTPRoute → Service → Pod`

Cilium может быть implementation этого Gateway:
`Gateway API objects → Cilium → Envoy/eBPF → workload`

Зачем: не обязательно отдельно ставить NGINX Ingress только ради HTTP ingress; Gateway API также богаче старого Kubernetes `Ingress`.
Cilium интегрирует Gateway API со своим networking/security stack.

## 15. Service Mesh
Service Mesh решает задачи service-to-service communication:

1. mTLS/encryption.
2. Identity.
3. L7 routing.
4. Traffic management.
5. Observability.
6. Security policy.

Классический Istio часто выглядит:

`App → Envoy sidecar → network → Envoy sidecar → App`

Cilium старается часть функций делать:

`App → eBPF → network`

а Envoy подключать там, где действительно требуется L7.

Зачем: потенциально меньше sidecar overhead и одна networking/security платформа вместо нескольких отдельных систем. Cilium официально объединяет Service Mesh с Gateway API, L7 policy, encryption и observability.

## 16. Observability

Observability ≠ просто metrics.

Она отвечает на вопросы:

```text
Кто отправил traffic?
Куда?
Какой port/protocol?
Был ли DROP?
Какая policy его заблокировала?
Какой DNS запрос?
Какой HTTP request?
```

Cilium/Hubble также интегрируется с Prometheus/Grafana для обычных metrics.

Это принципиальное отличие от Flannel, где сетевой troubleshooting в основном строится вокруг logs/routes/tcpdump.

## 17. Проверка

```bash
# Общий health
cilium status

# Полный автоматический network test
cilium connectivity test

# Agent status
kubectl -n kube-system exec ds/cilium -- \
cilium-dbg status

# Cilium endpoints
kubectl -n kube-system exec ds/cilium -- \
cilium-dbg endpoint list

# Services
kubectl -n kube-system exec ds/cilium -- \
cilium-dbg service list
```

## 18. Debug

1. `cilium status` — все ли компоненты healthy.
2. `cilium connectivity test` — какой именно network path сломан.
3. `hubble observe --verdict DROPPED` — режет ли traffic policy.
4. `cilium-dbg service list` — знает ли Cilium Service/backends.
5. `cilium-dbg bpf lb list` — есть ли backend в BPF map.
6. `ip route` — routing.
7. Проверить VXLAN/Geneve firewall.
8. Проверить MTU.
9. Проверить NetworkPolicy/CiliumNetworkPolicy.

## 19. Когда выбирать

1. Новый Kubernetes cluster.
2. Хочется eBPF-first networking.
3. Хочется убрать kube-proxy.
4. Нужны сильные NetworkPolicy, включая L7/FQDN.
5. Нужна хорошая network observability.
6. Нужен Hubble CLI/UI и Service Map.
7. Планируется Gateway API.
8. Нужны Service Mesh возможности.
9. Нужен единый стек `CNI + Service LB + security + observability`.
10. Команда готова к большей сложности, чем у Flannel.

**Коротко:** `Cilium = eBPF-first CNI → Pod networking + Service LB + kube-proxy replacement + L3-L7 policy + Hubble + Gateway API + Service Mesh`.
