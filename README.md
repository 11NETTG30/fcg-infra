# fcg-infra

Repositório de infraestrutura do **FIAP Cloud Games (FCG)**.

Contém os arquivos Docker Compose e manifestos Kubernetes para orquestrar todos os microsserviços do ecossistema FCG.

---

## 📁 Estrutura do Repositório

```
fcg-infra/
├── compose-images/     ← Docker Compose usando imagens publicadas no GHCR
├── compose-builds/     ← Docker Compose com build local dos microsserviços
└── k8s/
    ├── namespace.yaml
    ├── ingress.yaml
    ├── rabbitmq/
    ├── mailpit/
    ├── fcg-users/      ← API + PostgreSQL
    ├── fcg-catalog/    ← API + PostgreSQL
    ├── fcg-payments/   ← API + PostgreSQL
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
127.0.0.1 users-api.fcg
127.0.0.1 catalog-api.fcg
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

| Serviço | URL |
|---|---|
| fcg-users (Swagger) | http://users-api.fcg/swagger/index.html |
| fcg-catalog (Swagger) | http://catalog-api.fcg/swagger/index.html |
| RabbitMQ Management | http://rabbitmq.fcg |
| Grafana | http://grafana.fcg (admin / admin) |
| Mailpit | http://mailpit.fcg |

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
