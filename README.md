# PoC: Cilium eBPF como CNI e Security Policy Engine no K3s

PoC para validar controle de egress com Cilium em Kubernetes.

Escopo:

- FQDN para destinos externos com IP dinâmico
- `serverNames` para controle por SNI em TLS
- `toCIDR` para liberação direta por IP/porta

## Arquitetura do projeto

Estrutura do projeto:

```
.
├── README.md
├── apps/
│   ├── app-1.yaml              # Pod de teste 1 (ex: netshoot)
│   └── app-2.yaml              # Pod de teste 2 (ex: netshoot)
├── namespaces/
│   └── cilium-poc.yaml         # Namespace dedicado para a PoC
└── policies/
    ├── app-1-github-only.yaml  # Libera tráfego apenas para *.github.com via FQDN
    ├── app-2-httpbin-only.yaml  # Libera tráfego HTTPS para httpbin.org via TLS SNI
    └── app-2-ssh-only.yaml      # Libera SSH da app-2 para um IP fixo
```

## Por que Cilium

Comparação resumida:

- Cilium aplica policy direto no datapath com eBPF
- Istio adiciona sidecar e gateway para resolver o mesmo fluxo
- Cilium encaixa melhor quando a meta é egress simples e previsível
- Istio faz mais sentido quando a meta é malha completa e terminação centralizada

## Cilium x Istio egress gateway

Use Cilium quando quiser:

- reduzir componentes
- manter L3/L4 e SNI na própria policy
- evitar mistura entre roteamento, proxy e autorização
- tratar SSH, FQDN e TLS SNI com regras separadas

Use Istio egress gateway quando precisar de:

- terminação TLS centralizada
- inspeção L7 profunda
- roteamento avançado de malha
- padronização de tráfego de saída em um gateway único

## Padrões de policy

### FQDN

- indicado para destinos externos com IP variável
- depende do DNS do pod
- pode liberar IP compartilhado por outros hosts

### `serverNames`

- indicado para HTTPS com SNI visível
- distingue hosts no handshake TLS
- evita o problema de cache por IP do FQDN
- não substitui inspeção HTTP dentro do TLS

### `toCIDR`

- indicado para IP fixo
- útil para SSH e outros protocolos sem hostname
- simples e previsível

## Limitações

- `serverNames` depende de SNI
- FQDN continua sujeito a compartilhamento de IP
- inspeção de `Host` e path em HTTPS exige L7/proxy
- ECH/ESNI podem reduzir visibilidade do SNI

## L7 (HTTP/HTTPS) — Opções e requisitos

- O que o L7 consegue: para HTTP em claro (porta 80) o Cilium consegue inspecionar e filtrar por `method`, `path` e `headers` (incluindo `Host`). Isso funciona sem terminação TLS porque o proxy vê o tráfego em texto.
- HTTPS: existem duas opções principais:
    - `serverNames` (SNI): aplica-se no ClientHello do TLS e permite distinguir destinos pelo nome sem descriptografar o tráfego. Não vê `Host` HTTP nem path.
    - Terminação/interceptação TLS (L7 proxy): se o proxy (Envoy/Cilium) terminar o TLS, então o tráfego é descriptografado e as regras HTTP (Host, path, method, headers) podem ser aplicadas.
- Requisitos mínimos para usar L7 com inspeção HTTPS (terminação):
    - Cilium deve ter o suporte a L7/Envoy ativado (feature proxy/L7 habilitada na instalação).
    - Segredos de TLS (certificado/chave/CA) precisam estar disponíveis para o proxy terminar TLS — geralmente via Kubernetes Secret e com as flags de sincronização de policy-secret do Cilium se aplicável.
    - Considerar confiança/CA: interceptar TLS externo exige que clientes confiem no certificado/CA usado pelo proxy (complexo para destinos públicos).
- Limitações operacionais e riscos:
    - Terminar TLS aumenta complexidade operacional, exigindo gestão de certificados e possíveis alterações na confiança dos clientes.
    - CDN e IPs compartilhados continuam a criar colisões quando se usa apenas `toFQDN`.
    - Tecnologias que ocultam SNI (ECH/ESNI) reduzem eficácia de `serverNames`.

Próximo passo: para testes locais é simples criar uma policy L7 para HTTP (ex.: bloquear por `Host`/`path`) — para HTTPS, preparar a parte de certificados e as flags do Cilium antes de tentar a terminação.

Exemplos de policies L7 (exemplos completos abaixo):

HTTP L7 (inspeção de header `Host`, method e path):

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
    name: app-2-http-l7
    namespace: cilium-poc
spec:
    endpointSelector:
        matchLabels:
            app: app-2
    egress:
    - toPorts:
        - ports:
            - port: "80"
                protocol: TCP
            rules:
                http:
                - method: "GET"
                    path: "/status"
                - headers:
                    - name: "Host"
                        exact: "example.com"
```

HTTPS L7 com terminação (requere secret/terminação TLS):

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
    name: app-2-https-l7-terminate
    namespace: cilium-poc
spec:
    endpointSelector:
        matchLabels:
            app: app-2
    egress:
    - toPorts:
        - ports:
            - port: "443"
                protocol: TCP
            terminatingTLS:
                secret:
                    name: externaltarget-tls
            rules:
                http:
                - method: "GET"
                    path: "/anything"
                - headers:
                    - name: "Host"
                        exact: "httpbin.org"
```

Policy única com TLS termination para liberar por `Host`:

- Use `terminatingTLS` no `toPorts` da porta `443` e coloque a regra HTTP com `host: "api.github.com"`.
- Crie um Secret com a chave e o certificado que o Envoy vai apresentar ao pod. Para a policy acima, o Secret usado é `github-egress-tls`.
- Habilite o uso de Secrets de policy no Cilium para evitar leitura ampla do cluster:

```bash
cilium config set enable-l7-proxy true
cilium config set enable-policy-secrets-sync true
cilium config set policy-secrets-only-from-secrets-namespace true
cilium config set policy-secrets-namespace cilium-secrets
```

- Se o destino for público, você só consegue "garantir" o certificado se controlar a CA confiada pelo cliente ou se o alvo já for o seu próprio endpoint. Para GitHub público, essa abordagem só faz sentido em laboratório ou com proxy/CA sob seu controle.

## Instalação

Ambiente de referência: Ubuntu 24.04.

Para evitar conflitos de DNS entre o eBPF do Cilium e o `systemd-resolved` do hospedeiro, usamos a seguinte receita.

### 1) Instalação do K3s (sem CNI padrão)

Subimos o K3s desabilitando o Flannel e o `NetworkPolicy` nativo, preparando o terreno para o Cilium assumir a rede.

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--flannel-backend=none --disable-network-policy' sh -
```

### 2) Configuração do kubeconfig

Para operar o cluster sem `sudo`:

```bash
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG=~/.kube/config
```

### 3) Instalação do Cilium (versão recomendada: 1.19.5+)

Nota: use Cilium 1.19.5 ou superior.

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

## Validação

Com o Cilium operante, aplique apps e policies:

### Subir o ambiente

```bash
kubectl apply -f namespaces/cilium-poc.yaml
kubectl apply -f apps/
kubectl apply -f policies/
```

### Testes de conectividade

```bash
# Deve retornar HTTP 200
kubectl exec -it deploy/app-1 -n cilium-poc -- curl -I https://api.github.com

# Deve falhar/timeout
kubectl exec -it deploy/app-1 -n cilium-poc -- curl -m 5 -I https://httpbin.org
```
