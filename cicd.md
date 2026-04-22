# ⚙️ Pipelines CI/CD com GitHub Actions

## O que é CI/CD?

**CI (Continuous Integration)** é a prática de integrar e validar código automaticamente a cada mudança.
**CD (Continuous Delivery/Deployment)** é a prática de entregar essas mudanças automaticamente para o ambiente de produção.

Neste projeto, os dois pipelines cobrem desde a criação da infraestrutura até o deploy da aplicação.

---

## Pipeline 1 — Terraform CI/CD

### Trigger
Acionado manualmente via `workflow_dispatch`, com três opções controláveis:

```yaml
on:
  workflow_dispatch:
    inputs:
      apply:
        description: 'Executar o terraform apply?'
        type: choice
        options: ["true", "false"]

      plan_destroy:
        description: 'Planejar o terraform destroy?'
        type: choice
        options: ["true", "false"]

      destroy:
        description: 'Executar o terraform destroy?'
        type: choice
        options: ["true", "false"]
```

### Etapas do pipeline

```
Checkout → Configure AWS (OIDC) → Setup SSH Key → Setup Terraform
    → Init → Validate → Plan → Apply (condicional) → Destroy (condicional)
```

### Controle condicional de steps

```yaml
- name: Terraform Apply
  run: terraform apply -auto-approve tfplan.tfout
  if: github.event.inputs.apply == 'true'

- name: Terraform Destroy
  run: terraform apply -destroy -auto-approve tfplan.tfout
  if: github.event.inputs.destroy == 'true'
```

---

## Pipeline 2 — Deploy CI/CD

### Trigger
Acionado automaticamente a cada `push` na branch `main`, simulando o fluxo de uma equipe de desenvolvimento.

### Etapas do pipeline

```
Checkout → Configure AWS (OIDC) → Build Docker image
    → Push para ECR → SSH na EC2 → docker pull + docker run
```

### Deploy na EC2 via SSH

```bash
echo "$PRIVATE_KEY" > site-prod-key
chmod 400 site-prod-key

ssh -i site-prod-key -o StrictHostKeyChecking=no ec2-user@$PUBLIC_IP << EOF
  docker stop site-prod || true
  docker rm site-prod || true
  aws ecr get-login-password --region us-east-2 | \
    docker login --username AWS --password-stdin 324793865887.dkr.ecr.us-east-2.amazonaws.com
  docker pull 324793865887.dkr.ecr.us-east-2.amazonaws.com/site_prod:v1.0
  docker run -d -p 80:80 --name site-prod \
    324793865887.dkr.ecr.us-east-2.amazonaws.com/site_prod:v1.0
EOF

rm site-prod-key
```

---

## Autenticação OIDC

Ambos os pipelines usam OIDC para assumir uma IAM Role na AWS sem precisar armazenar credenciais como secrets estáticos.

```yaml
permissions:
  id-token: write
  contents: read

- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::324793865887:role/GitHub-Infra-Role
    aws-region: us-east-2
```

A permissão `id-token: write` é obrigatória para que o GitHub gere o token OIDC.

---

## Secrets necessários no GitHub

| Secret | Descrição |
|---|---|
| `TF_PUBLIC_KEY` | Chave pública SSH para cadastrar no EC2 |
| `TF_PRIVATE_KEY` | Chave privada SSH para conectar na EC2 |

> Os secrets são configurados em **Settings → Secrets and variables → Actions** em cada repositório.

---

## Boas práticas aprendidas

- Usar OIDC em vez de access keys estáticos — mais seguro e sem rotação manual
- O nome do secret no GitHub deve ser **idêntico** ao usado no workflow
- Sempre adicionar `chmod 400` na chave privada antes de usar SSH
- Usar `docker stop || true` antes do `docker run` para evitar conflito de nomes de container
- Usar `workflow_dispatch` para operações destrutivas como `terraform destroy` (OPCIONAL, pois foi colocado pra testar a criação do container na ECR)
---

[Go to Previous](terraform.md)  | [Go to Next](docker-ecr.md)
