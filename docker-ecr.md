# 🐳 Docker + ECR

## O que é Docker?

Docker é uma plataforma de **containerização** que empacota a aplicação e todas as suas dependências em um container isolado. Isso garante que a aplicação rode de forma idêntica em qualquer ambiente — desenvolvimento, CI/CD ou produção.

---

## O que é ECR?

O **Elastic Container Registry (ECR)** é o serviço da AWS para armazenar imagens Docker de forma privada e segura, integrado nativamente com outros serviços AWS como EC2 e ECS.

---

## Fluxo de imagens no projeto

```
Dockerfile (local/CI)
      │
      │ docker build
      ▼
Imagem Docker local
      │
      │ docker push
      ▼
ECR (site_prod:v1.0)
      │
      │ docker pull (na EC2)
      ▼
Container rodando na EC2 (porta 80)
      │
      │ http
      ▼
Usuário acessa o site
```

---

## Autenticação no ECR

Para fazer push ou pull de imagens no ECR, é necessário autenticar o Docker com as credenciais AWS:

```bash
aws ecr get-login-password --region us-east-2 | \
  docker login --username AWS --password-stdin \
  <id_da_conta_aws>.dkr.ecr.us-east-2.amazonaws.com
```

---

## Comandos utilizados no projeto

```bash
# Build da imagem
docker build -t site_prod:v1.0 .

# Tag para o ECR
docker tag site_prod:v1.0 \
  <id_da_conta_aws>.dkr.ecr.us-east-2.amazonaws.com/site_prod:v1.0

# Push para o ECR
docker push \
  <id_da_conta_aws>.dkr.ecr.us-east-2.amazonaws.com/site_prod:v1.0

# Pull na EC2
docker pull \
  <id_da_conta_aws>.dkr.ecr.us-east-2.amazonaws.com/site_prod:v1.0

# Rodar o container
docker run -d -p 80:80 --name site-prod \
  <id_da_conta_aws>.dkr.ecr.us-east-2.amazonaws.com/site_prod:v1.0
```

---

## IAM Role na EC2

A instância EC2 tem a role `EC2-ECR-Role` associada, o que permite fazer `docker pull` do ECR sem precisar de credenciais explícitas. A role precisa ter a policy `AmazonEC2ContainerRegistryReadOnly`.

---

## Recurso Terraform para o ECR

```hcl
resource "aws_ecr_repository" "ecr-site" {
  name         = "site_prod"
  force_delete = true
}
```

> `force_delete = true` é essencial — sem ele, o `terraform destroy` falha se o repositório tiver imagens armazenadas.

---

## Boas práticas aprendidas

- Sempre usar `docker stop && docker rm` antes de um novo `docker run` para evitar conflito de nomes
- A EC2 precisa de uma IAM Role com permissão de leitura no ECR para fazer pull sem credenciais
- Usar `force_delete = true` no recurso ECR do Terraform para facilitar o destroy
- Acessar a aplicação sempre via `http://` — sem SSL configurado, o `https://` é recusado pelo navegador