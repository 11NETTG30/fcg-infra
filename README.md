# FIAP Cloud Games (FCG) - Repositório de Orquestração - fcg-infra

## 📚 Sobre o Projeto

Este repositório faz parte do **Tech Challenge da Pós-Graduação em Arquitetura de Sistemas .NET da FIAP**, **Turma 11NETT – Grupo 30**.

Este é o repositório de orquestração dos microsserviços.

O objetivo do projeto é a construção de uma **plataforma de games educacionais**, chamada **FIAP Cloud Games (FCG)**, voltada para o aprendizado e prática de conceitos de tecnologia, utilizando boas práticas de arquitetura de software.

---

## 🚀 Setup Inicial

### 1. Comandos Docker

```bash
# Criar personal access token no Github para acessar repositório:
URL da documentação: https://docs.github.com/pt/packages/working-with-a-github-packages-registry/working-with-the-container-registry

Acesse a URL: https://github.com/settings/tokens/new?scopes=write:packages

# Logar no serviço do Container Registry:

Opção 1:
Salve o token como variável de ambiente:

Linux:
export CR_PAT=YOUR_TOKEN
Windows (temporário):
set CR_PAT YOUR_TOKEN
Windows (permanente):
setx CR_PAT YOUR_TOKEN

Entrar no serviço do Container registry:
echo $CR_PAT | docker login ghcr.io -u USERNAME --password-stdin


OU, Opção 2:
Faça login no registry pelo Developer PowerShell:

docker login ghcr.io -u NOMEUSUARIO
e digitar o token obtido anteriormente.

#Fazer o build da imagem:
docker build -t ghcr.io/11nettg30/payment-api:latest -f src/API/Dockerfile src

# Fazer push da imagem:
docker push ghcr.io/11nettg30/payment-api:latest

# Verificar se a imagem subiu corretamente:
docker inspect ghcr.io/11nettg30/payment-api:latest
```

### 2. Comandos Kubernetes
```bash
#Aplicar o namespace:
 kubectl apply -f .\namespace.yaml

#Subindo tudo 
Obs: Garanta que as imagens dos microsserviços estão públicas (https://github.com/orgs/11NETTG30/packages):
 kubectl apply -f .\k8s\rabbitmq\
 kubectl apply -f .\k8s\fcg-payments\postgres\
 kubectl apply -f .\k8s\fcg-payments\API\

#Subir novamente um serviço:
 kubectl rollout restart deployment payment-api -n fcg

#Verificar Pods:
 kubectl get pods -n fcg

#Verificar services:
 kubectl get svc -n fcg

#Para deletar um pod:
 kubectl delete pod postgres-payment-6645995bb5-wlvdd -n fcg

#Para listar deployments
 kubectl get deployments -n fcg

#Para deletar deployments
 kubectl delete deployment postgres-payment -n fcg

#Para checar os logs de um pod:
 kubectl logs postgres-payment-5f445b8b64-xdm6w -n fcg

#Para deletar tudo:
 kubectl delete all --all -n fcg
```