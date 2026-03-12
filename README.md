# FIAP Cloud Games (FCG) - Repositório de Orquestração - fcg-infra

## 📚 Sobre o Projeto

Este repositório faz parte do **Tech Challenge da Pós-Graduação em Arquitetura de Sistemas .NET da FIAP**, **Turma 11NETT – Grupo 30**.

Este é o repositório de orquestração dos microsserviços.

O objetivo do projeto é a construção de uma **plataforma de games educacionais**, chamada **FIAP Cloud Games (FCG)**, voltada para o aprendizado e prática de conceitos de tecnologia, utilizando boas práticas de arquitetura de software.

---

## 🚀 Setup Inicial

### 1. Configurar Variáveis de Ambiente do docker-compose

```bash
# Copie o arquivo de exemplo
cp .env.example .env

# Edite o .env com suas credenciais
```

### 2. Configurar Variáveis de Ambiente dos projetos

```bash
# Copie o arquivo de exemplo de cada projeto
cp .env.example .env

# Edite o .env com suas credenciais
```

### 3. Comandos Docker

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

# Executar o docker-compose
docker-compose up --build -d

# Derrubar containers:
docker-compose down
```