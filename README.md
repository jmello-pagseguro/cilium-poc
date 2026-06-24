# PoC: Cilium eBPF como CNI e Security Policy Engine no K3s

Esta Prova de Conceito (PoC) demonstra a substituição da infraestrutura de rede tradicional (IPTables/Flannel) pelo Cilium, utilizando eBPF para roteamento de alta performance e controle de tráfego de saída (Egress) baseado em FQDN (Fully Qualified Domain Name).

## Arquitetura do projeto

A estrutura de diretórios foi organizada para separar a infraestrutura lógica, as aplicações de teste e as políticas de segurança:

```
.
├── README.md
├── apps/
│   ├── app-1.yaml              # Pod de teste 1 (ex: netshoot)
│   └── app-2.yaml              # Pod de teste 2 (ex: netshoot)
├── namespaces/
│   └── cilium-poc.yaml         # Namespace dedicado para a PoC
└── policies/
        ├── app-1-github-only.yaml  # Libera tráfego apenas para *.github.com
        └── app-2-httpbin-only.yaml # Libera tráfego apenas para *.httpbin.org
```

## Por que Cilium e não Istio? (eBPF vs Sidecar)

Para resolver o desafio de controle de tráfego e segurança, o ecossistema Kubernetes tradicionalmente recorre ao Istio. No entanto, esta PoC valida o Cilium como uma alternativa moderna por diferenças fundamentais na arquitetura:

**O Modelo Istio (Sidecar):** o Istio injeta um contêiner proxy (Envoy) dentro de cada pod da aplicação. Todo pacote de rede precisa sair da aplicação, passar pelas regras de IPTables do Linux, entrar no Envoy, ser processado, voltar pro IPTables e só então sair para a rede. Isso gera um grande overhead de CPU, memória e latência.

**O Modelo Cilium (eBPF / Sidecarless):** o Cilium roda a nível de kernel (eBPF). Ele não injeta proxies em cada pod para tarefas de L3/L4; os pacotes são interceptados diretamente na interface de rede virtual do pod com performance nativa.

## Escopo atual — Controle FQDN via DNS (L4)

Atualmente, esta PoC implementa restrição de saída baseada em FQDN operando na camada 4.

**Como funciona:** o Cilium intercepta as requisições DNS (porta 53) dos pods, lê as respostas e armazena os IPs resolvidos (ex.: os IPs dinâmicos da api.github.com) em um cache. Em seguida, ele libera a porta 443 apenas para esses IPs em L4.

**Gap de segurança (Spoofing de IP/CDN):** se `api.github.com` e um domínio malicioso `hacker.github.io` compartilharem o mesmo IP de um provedor de CDN, a liberação por L4 permitirá o tráfego para ambos, pois o IP de destino é o mesmo.

**Próximos passos (L7):** para fechar esse gap e validar a URL exata independentemente do IP, o Cilium possui integração nativa com o Envoy para realizar inspeção HTTP (camada 7) via cabeçalho `Host`. Isso será explorado na próxima fase da PoC.

## Passo a passo de instalação (Ubuntu 24.04 compatível)

O ambiente de testes foi configurado no Ubuntu 24.04. Para evitar conflitos de DNS conhecidos entre o eBPF do Cilium e o `systemd-resolved` do hospedeiro, utilizamos a seguinte receita de instalação "blindada".

### 1) Instalação do K3s (sem CNI padrão)

Subimos o K3s desabilitando o Flannel e o `NetworkPolicy` nativo, preparando o terreno para o Cilium assumir a rede.

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--flannel-backend=none --disable-network-policy' sh -
```

### 2) Configuração do kubeconfig

Para garantir que o seu usuário tenha as permissões corretas para operar o cluster sem uso de `sudo`:

```bash
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG=~/.kube/config
```

### 3) Instalação do Cilium (versão recomendada: 1.19.5+)

Nota: é crucial usar a versão 1.19.5 ou superior para garantir as correções de compatibilidade com o Ubuntu 24.04.

Instalamos o Cilium substituindo o kube-proxy nativo, mas protegendo explicitamente os serviços do hospedeiro (`hostServices.enabled=false`) para não derrubar o DNS da máquina física.

```bash
cilium install --version 1.19.5 \
    --set ipv6.enabled=false \
    --set kubeProxyReplacement=true \
    --set hostServices.enabled=false \
    --set localRedirectPolicy=false

# aguardar inicialização
cilium status --wait
```

## Validando a PoC

Com o Cilium operante, aplique as aplicações e as políticas para testar o filtro.

### Subir o ambiente

```bash
kubectl apply -f namespaces/cilium-poc.yaml
kubectl apply -f apps/
kubectl apply -f policies/
```

### Testes de conectividade (dentro do namespace `cilium-poc`)

```bash
# Deve retornar HTTP 200 (app-1 possui permissão para *.github.com)
kubectl exec -it deploy/app-1 -n cilium-poc -- curl -I https://api.github.com

# Deve falhar/timeout para destinos não permitidos
kubectl exec -it deploy/app-1 -n cilium-poc -- curl -m 5 -I https://httpbin.org
```
