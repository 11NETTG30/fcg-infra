# fcg-infra

Repositório de infraestrutura do **FIAP Cloud Games (FCG)**.

Contém os arquivos Docker Compose e manifestos Kubernetes para orquestrar todos os microsserviços do ecossistema FCG.

---

## 📁 Estrutura do Repositório

```
fcg-infra/
├── .github/
│   └── workflows/
│       └── kong-publish.yml   ← CI para build e push da imagem customizada do Kong
├── compose-images/            ← Docker Compose usando imagens publicadas no GHCR
├── compose-builds/            ← Docker Compose com build local dos microsserviços
├── kong/
│   └── Dockerfile             ← Imagem customizada do Kong com plugin jwt-keycloak
└── k8s/
    ├── namespace.yaml
    ├── ingress.yaml
    ├── kong/                  ← API Gateway
    │   ├── configmap.yaml     ← Configuração declarativa (rotas, plugins, JWT)
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── secret.example.yaml
    ├── rabbitmq/
    ├── mailpit/
    ├── fcg-users/             ← API + PostgreSQL
    ├── fcg-catalog/           ← API + PostgreSQL
    ├── fcg-payments/          ← API + PostgreSQL
    ├── fcg-notifications/
    └── observabilidade/
        ├── grafana/
        ├── loki/
        ├── otel-collector/
        ├── prometheus/
        └── tempo/
```

---

## 🔧 Serviços de Suporte

### Observabilidade

| Serviço | Descrição |
|---|---|
| **OTel Collector** | Ponto central de coleta de telemetria. Recebe métricas, logs e traces dos microsserviços via protocolo OTLP e os encaminha para os backends apropriados (Prometheus, Loki e Tempo). Desacopla os serviços das ferramentas de observabilidade — se trocar o backend, só muda a configuração do Collector |
| **Prometheus** | Banco de dados de séries temporais para métricas. Coleta e armazena dados como latência de requisições, taxa de erros e uso de recursos. Base para os painéis de métricas no Grafana |
| **Loki** | Agregador de logs. Indexa e armazena os logs de todos os serviços de forma eficiente, permitindo buscas por texto e por labels (ex: serviço, nível de log). Integrado ao Grafana para consulta |
| **Tempo** | Backend de tracing distribuído. Armazena os traces gerados pelos microsserviços, permitindo visualizar o caminho completo de uma requisição passando por múltiplos serviços e identificar gargalos. Integrado ao Grafana para exploração |
| **Grafana** | Interface de visualização unificada para toda a stack de observabilidade. Reúne em um único lugar dashboards de métricas (Prometheus), busca de logs (Loki) e exploração de traces (Tempo) |

### Mailpit

Servidor SMTP para desenvolvimento. Captura todos os e-mails enviados pelos microsserviços e os exibe em uma interface web, sem precisar de uma conta de e-mail real nem de configuração de SMTP externo. Útil para validar os e-mails disparados pelo `fcg-notifications`.

Acessível em `http://mailpit.fcg` (Kubernetes) ou `http://localhost:8025` (Docker Compose).

---

## 📋 Pré-requisitos

- [Docker](https://www.docker.com/) + Docker Compose
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Minikube](https://minikube.sigs.k8s.io/) (ou Kubernetes via Docker Desktop)
- Acesso ao GitHub Container Registry (GHCR) da organização [11NETTG30](https://github.com/orgs/11NETTG30/packages)

---

## 🐳 Docker Compose

Existem dois ambientes Docker Compose com propósitos distintos:

| Pasta | Descrição |
|---|---|
| `compose-images/` | Sobe o ambiente completo usando as imagens já publicadas no GHCR |
| `compose-builds/` | Faz o build local de cada microsserviço a partir do código-fonte |

### Portas expostas (Docker Compose)

| Serviço | Porta |
|---|---|
| fcg-users | `5083` |
| fcg-catalog | `5084` |
| fcg-payments | `5085` |
| RabbitMQ Management | `15672` |
| Mailpit (UI) | `8025` |

### 1. Configurar variáveis de ambiente

Copie o arquivo de exemplo e preencha os valores:

```bash
# compose-images
cp compose-images/.env.example compose-images/.env

# compose-builds
cp compose-builds/.env.example compose-builds/.env
```

> O `compose-builds/.env` tem um campo extra: `NUGET_AUTH_TOKEN` — PAT do GitHub com escopo `read:packages`, necessário para restaurar os pacotes NuGet do [fcg-shared](https://github.com/11NETTG30/fcg-shared).

### 2. Subir o ambiente

```bash
# Usando imagens publicadas
docker compose -f compose-images/docker-compose.yml up -d

# Ou fazendo build local (aproveita cache se as imagens já existem)
docker compose -f compose-builds/docker-compose.yml up -d

# Rebuild obrigatório (use após alterações no código-fonte)
docker compose -f compose-builds/docker-compose.yml up -d --build
```

### Comandos úteis

```bash
# Parar
docker compose -f compose-images/docker-compose.yml down

# Parar e remover volumes (apaga dados dos bancos)
docker compose -f compose-images/docker-compose.yml down -v
```

---

## ☸️ Kubernetes

### Pré-requisitos

**Minikube:**

```bash
# Habilitar o Ingress Controller
minikube addons enable ingress

# Iniciar o tunnel (manter o terminal aberto)
minikube tunnel
```

**Docker Desktop:**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

### Configurar o arquivo hosts

Adicione as entradas abaixo no arquivo hosts do sistema operacional:
- **Linux / macOS:** `/etc/hosts`
- **Windows:** `C:\Windows\System32\drivers\etc\hosts`

```
127.0.0.1 api.fcg
127.0.0.1 rabbitmq.fcg
127.0.0.1 grafana.fcg
127.0.0.1 mailpit.fcg
```

### Configurar secrets

Cada serviço possui um `secret.example.yaml` com os campos necessários. Copie e preencha antes de aplicar:

```bash
cp k8s/rabbitmq/secret.example.yaml           k8s/rabbitmq/secret.yaml
cp k8s/fcg-users/API/secret.example.yaml      k8s/fcg-users/API/secret.yaml
cp k8s/fcg-users/postgres/secret.example.yaml k8s/fcg-users/postgres/secret.yaml
# ... repita para os demais serviços
```

### Subir o ambiente

```bash
# Aplicar namespace (primeira vez)
kubectl apply -f k8s/namespace.yaml

# Garantir que as imagens dos microsserviços estão públicas no GHCR:
# https://github.com/orgs/11NETTG30/packages

# Aplicar todos os manifestos recursivamente
kubectl apply -f k8s/ --recursive
```

### URLs de acesso

| Serviço | URL | Observação |
|---|---|---|
| API Gateway | http://api.fcg | Ponto único de entrada — todas as requisições passam pelo Kong |
| RabbitMQ Management | http://rabbitmq.fcg | — |
| Grafana | http://grafana.fcg | admin / admin |
| Mailpit | http://mailpit.fcg | — |

### Swagger (documentação das APIs)

O Swagger não é exposto pelo Kong. Para acessá-lo, utilize port-forward direto no serviço:

```bash
kubectl port-forward svc/users-api 5083:80 -n fcg
kubectl port-forward svc/catalog-api 5084:80 -n fcg
```

| Serviço | URL |
|---|---|
| fcg-users | http://localhost:5083/swagger/index.html |
| fcg-catalog | http://localhost:5084/swagger/index.html |

> ⚠️ O "Try it out" do Swagger bate diretamente na API via port-forward,
> sem passar pelo Kong. Use-o apenas para consultar as rotas disponíveis.
> Para testar o roteamento pelo Kong, utilize o Postman ou similar.

### Atualizar um serviço

Após publicar uma nova imagem com a mesma tag (ex: `:latest`), o `apply` sozinho não recria os pods pois o manifesto não mudou. Use:

```bash
kubectl rollout restart deployment <nome-do-deployment> -n fcg
```

### Comandos de diagnóstico

```bash
# Listar pods
kubectl get pods -n fcg

# Listar services
kubectl get svc -n fcg

# Listar deployments
kubectl get deployments -n fcg

# Logs de um pod
kubectl logs <nome-do-pod> -n fcg

# Entrar em um pod
kubectl exec -it <nome-do-pod> -n fcg -- bash

# Reiniciar um deployment
kubectl rollout restart deployment <nome-do-deployment> -n fcg

# Conectar em um banco de dados:
kubectl port-forward svc/mongodb-catalog -n fcg 27017:27017

```

### Limpeza

```bash
# Remover todos os recursos do namespace
kubectl delete all --all -n fcg

# Remover PVCs dos bancos de dados
kubectl delete pvc -n fcg --all
```

---

## Decisões Arquiteturais e Riscos

### JWT Keycloak Plugin (Kong)

**Decisão:** Utilização do plugin `jwt-keycloak` via fork da Platformatory
em vez do plugin `jwt` nativo do Kong.

**Motivo:** O plugin nativo exige que a chave pública RSA seja declarada
estaticamente no configmap, o que significa que toda vez que a chave rotacionar
na UsersAPI, o configmap precisaria ser atualizado manualmente e o Kong
reiniciado. O `jwt-keycloak` resolve isso via JWKS discovery, buscando
a chave pública diretamente no endpoint `/.well-known/jwks.json` da UsersAPI
de forma automática.

**Riscos:**
- O plugin original foi abandonado em 2021. O fork utilizado é mantido
  pela Platformatory para uso interno, sem garantias de suporte.
- Testado oficialmente até Kong 3.4. O ambiente utiliza Kong 3.9 —
  compatibilidade não garantida.
- Dependência de repositório externo no build da imagem. Se o repositório
  for removido ou alterado, o CI quebra.
- Fork com baixa adoção (4 estrelas no GitHub), o que reduz a chance de
  bugs conhecidos serem reportados e corrigidos.

**Importante:** Esta abordagem foi adotada como exercício acadêmico para
explorar o conceito de JWKS discovery e rotação automática de chaves.
Em um ambiente real de produção, esta solução provavelmente não seria
adotada. As alternativas mais adequadas seriam:
- **Kong Enterprise** com o plugin `openid-connect` nativo e suportado oficialmente.
- **Plugin `jwt` nativo** com um processo automatizado de rotação de chave
  via CI/CD, que atualizaria o configmap e reiniciaria o Kong sem intervenção manual.
- **Migração para um provedor de identidade dedicado** como Keycloak ou Auth0,
  que possuem integrações oficiais e suportadas com o Kong.

**Alternativa considerada:** Plugin `jwt` nativo com chave pública estática
no configmap. Mais simples e sem dependências externas, porém exige
intervenção manual a cada rotação de chave.