# 🏗️ Arquitetura do Projeto

## Visão Geral

A infraestrutura foi projetada com dois fluxos independentes: um para provisionar os recursos na AWS e outro para fazer o deploy da aplicação. Essa separação garante que mudanças na infraestrutura e mudanças no código da aplicação sejam gerenciadas de forma independente.

---

## Fluxo 1 — Provisionamento de Infraestrutura

```

![Fluxo Terraform](fluxo-terraform.svg)

```

O pipeline suporta três operações controladas por `workflow_dispatch`:

| Input | Descrição |
|---|---|
| `apply` | Cria ou atualiza a infraestrutura |
| `plan_destroy` | Visualiza o que será destruído |
| `destroy` | Remove toda a infraestrutura |

---

## Fluxo 2 — Deploy da Aplicação

```

![Fluxo Deploy](fluxo-deploy.svg)

```

---

## Recursos AWS Provisionados

### EC2 — website-server
- **AMI:** Amazon Linux 2 (`ami-0a1b6a02658659c2a`)
- **Tipo:** t3.micro
- **Região:** us-east-2
- **IP público:** associado automaticamente
- **IAM Role:** EC2-ECR-Role (permite pull de imagens no ECR)

### ECR — site_prod
- Repositório privado de imagens Docker
- `force_delete = true` para permitir destroy mesmo com imagens

### S3 — terraform state
- Armazena o arquivo `.tfstate` remotamente
- Garante que o estado da infraestrutura seja compartilhado e versionado

### Security Group — website-sg
| Tipo | Porta | Origem |
|---|---|---|
| SSH | 22 | 0.0.0.0/0 |
| HTTP | 80 | 0.0.0.0/0 |
| HTTPS | 443 | 0.0.0.0/0 |
| Saída | Todos | 0.0.0.0/0 |

---

## Autenticação — OIDC

Em vez de armazenar credenciais AWS como secrets no GitHub, o projeto utiliza **OpenID Connect (OIDC)**. O GitHub Actions assume uma IAM Role diretamente, sem precisar de `AWS_ACCESS_KEY_ID` ou `AWS_SECRET_ACCESS_KEY` estáticos.

A Trust Policy da IAM Role valida que a requisição vem do repositório correto:

```json
{
  "Condition": {
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:Edugon0/NOME-DO-REPO:*"
    }
  }
}
```
[Go to Previous](README.md)  | [Go to Next](terraform.md)
