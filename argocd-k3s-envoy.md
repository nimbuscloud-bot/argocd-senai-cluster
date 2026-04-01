# ArgoCD no k3s com Envoy Gateway — Rede Interna

> **Objetivo:** Instalar o k3s, subir o ArgoCD e expô-lo na rede interna via Kubernetes Gateway API (Envoy Gateway), acessível pelo domínio `argocd.study`.

---

## Pré-requisitos

- Servidor Ubuntu/Debian com IP fixo na rede (ex: `192.168.1.100`)
- Acesso root ou sudo
- Porta `80` liberada no firewall do servidor

---

## Visão geral da arquitetura

```
PC do usuário (Windows)
    │  http://argocd.study
    ▼
dnsmasq (servidor) → resolve argocd.study → 192.168.1.100
    ▼
Envoy Gateway (porta 80, LoadBalancer nativo do k3s)
    ▼
HTTPRoute → argocd-server:80
    ▼
ArgoCD UI
```

---

## Parte 1 — Instalar o k3s

### 1.1 Instalar o k3s sem o Traefik

O k3s vem com Traefik por padrão. Como vamos usar o Envoy Gateway, desabilitamos o Traefik na instalação:

```bash
curl -sfL https://get.k3s.io | sh -s - \
  --disable traefik \
  --write-kubeconfig-mode 644
```

### 1.2 Configurar o kubectl

```bash
mkdir -p ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
chmod 600 ~/.kube/config

# Opcional: adicionar ao .bashrc/.zshrc
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

### 1.3 Verificar o cluster

```bash
kubectl get nodes
# NAME       STATUS   ROLES                  AGE   VERSION
# servidor   Ready    control-plane,master   1m    v1.x.x
```

---

## Parte 2 — Instalar o ArgoCD

### 2.1 Criar o namespace e aplicar o manifesto

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2.2 Aguardar os pods subirem

```bash
kubectl wait --for=condition=Available \
  deployment/argocd-server \
  -n argocd \
  --timeout=120s
```

### 2.3 Desativar o TLS interno do ArgoCD

O Envoy Gateway vai terminar o TLS externamente. O ArgoCD precisa rodar em modo insecure:

```bash
kubectl patch configmap argocd-cmd-params-cm \
  -n argocd \
  --type merge \
  -p '{"data": {"server.insecure": "true"}}'

kubectl rollout restart deployment argocd-server -n argocd
```

### 2.4 Pegar a senha inicial do admin

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

> Guarde essa senha — ela será usada no primeiro login.

---

## Parte 3 — Instalar o Envoy Gateway

### 3.1 Instalar os CRDs do Kubernetes Gateway API

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

### 3.2 Instalar o Envoy Gateway via Helm

```bash
helm install eg \
  oci://docker.io/envoyproxy/gateway-helm \
  --version v1.2.0 \
  -n envoy-gateway-system \
  --create-namespace
```

### 3.3 Aguardar o controller

```bash
kubectl wait --timeout=5m \
  -n envoy-gateway-system \
  deployment/envoy-gateway \
  --for=condition=Available
```

---

## Parte 4 — Criar o Gateway e o HTTPRoute

### 4.1 GatewayClass e Gateway

Crie o arquivo `gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: argocd-gateway
  namespace: argocd
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same
```

```bash
kubectl apply -f gateway.yaml
```

### 4.2 HTTPRoute

Crie o arquivo `argocd-httproute.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: argocd-route
  namespace: argocd
spec:
  parentRefs:
    - name: argocd-gateway
      namespace: argocd
  hostnames:
    - "argocd.study"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: argocd-server
          port: 80
```

```bash
kubectl apply -f argocd-httproute.yaml
```

### 4.3 Verificar o status

```bash
kubectl get gateway -n argocd
kubectl get httproute -n argocd

# O Gateway deve mostrar PROGRAMMED = True
# O HTTPRoute deve mostrar Accepted = True
```

---

## Parte 5 — Instalar e configurar o dnsmasq

### 5.1 Instalar

```bash
sudo apt update && sudo apt install dnsmasq -y
```

### 5.2 Configurar

```bash
sudo nano /etc/dnsmasq.conf
```

Adicione ao final do arquivo:

```conf
# DNS upstream para internet
server=8.8.8.8
server=1.1.1.1

# Interface de rede local (ajuste conforme sua interface: ip a)
interface=eth0
bind-interfaces

# Entrada do ArgoCD — aponta para o IP do servidor
address=/argocd.study/192.168.1.100
```

> **Atenção:** substitua `eth0` pelo nome da sua interface de rede (`ip a` para verificar) e `192.168.1.100` pelo IP real do servidor.

### 5.3 Ativar e iniciar

```bash
sudo systemctl restart dnsmasq
sudo systemctl enable dnsmasq
```

### 5.4 Liberar a porta 53 no firewall

```bash
sudo ufw allow 53/udp
sudo ufw allow 53/tcp
sudo ufw allow 80/tcp
```

### 5.5 Testar resolução local

```bash
nslookup argocd.study 127.0.0.1
# Deve retornar 192.168.1.100
```

---

## Parte 6 — Configurar os clientes Windows

### Opção A — Manual (por máquina)

```
Painel de Controle
  → Central de Redes e Compartilhamento
  → Alterar configurações do adaptador
  → Propriedades do adaptador de rede
  → Protocolo TCP/IPv4
  → Servidor DNS preferencial: 192.168.1.100
```

### Opção B — Via DHCP do roteador (recomendado)

No painel do seu roteador, nas configurações de DHCP, defina:

```
DNS Primário: 192.168.1.100
```

Todos os dispositivos que pegarem IP via DHCP já usarão o dnsmasq automaticamente.

### Verificar do Windows

Abra o `cmd` e execute:

```cmd
nslookup argocd.study
```

Deve retornar `192.168.1.100`. Acesse no browser:

```
http://argocd.study
```

---

## Verificação final

```bash
# Pods do ArgoCD rodando
kubectl get pods -n argocd

# Gateway com IP externo atribuído
kubectl get svc -n envoy-gateway-system

# HTTPRoute aceito
kubectl describe httproute argocd-route -n argocd
```

---

## Resumo dos componentes

| Componente | Função |
|---|---|
| **k3s** | Kubernetes leve para servidores on-premise |
| **ArgoCD** | GitOps CD — interface exposta via HTTP |
| **Envoy Gateway** | Implementação do Kubernetes Gateway API |
| **Gateway + HTTPRoute** | Roteamento por hostname (`argocd.study`) |
| **dnsmasq** | Servidor DNS local que resolve `argocd.study` para o IP do servidor |
| **DHCP do roteador** | Distribui o dnsmasq como DNS para toda a rede |

---

## Próximos passos sugeridos

- **Adicionar TLS:** instalar o `cert-manager` com issuer self-signed e adicionar listener HTTPS no Gateway
- **Outros serviços:** qualquer novo serviço pode ser exposto adicionando um novo `HTTPRoute` com hostname diferente (ex: `grafana.study`) sem precisar mexer no DNS — basta adicionar `address=/grafana.study/192.168.1.100` no dnsmasq
- **Alta disponibilidade:** migrar para k3s multi-node adicionando workers com `K3S_URL` e `K3S_TOKEN`
