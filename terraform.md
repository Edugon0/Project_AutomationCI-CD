# 🌱 Terraform — O que Aprendi

## O que é Terraform?

Terraform é uma ferramenta de **Infraestrutura como Código (IaC)** da HashiCorp. Com ela, você descreve os recursos de infraestrutura que quer criar em arquivos `.tf` usando a linguagem HCL (HashiCorp Configuration Language), e o Terraform se encarrega de criar, atualizar ou destruir esses recursos na nuvem.

---

## Fluxo básico de trabalho

```
terraform init → terraform plan → terraform apply → terraform destroy
```

| Comando | O que faz |
|---|---|
| `terraform init` | Baixa os providers e inicializa o backend |
| `terraform plan` | Mostra o que será criado/modificado/destruído |
| `terraform apply` | Executa as mudanças na infraestrutura |
| `terraform destroy` | Remove todos os recursos criados |

---

## Estrutura dos arquivos

```
infra/
├── ec2.tf         ← instância EC2, key pair e security group
├── main.tf        ← recursos principais (ECR, S3)
├── variables.tf   ← declaração de variáveis
├── outputs.tf     ← valores que o Terraform exibe após o apply
├── provider.tf    ← configuração do provider AWS
└── user_data.sh   ← script executado na inicialização da EC2
```

---

## State file (tfstate)

O Terraform mantém um arquivo `.tfstate` que registra o estado atual da infraestrutura. Neste projeto, esse arquivo é armazenado remotamente no **S3**, o que garante:

- Estado compartilhado entre máquinas e pipelines
- Histórico de mudanças
- Segurança (não fica no repositório)

Configuração do backend S3:

```hcl
terraform {
  backend "s3" {
    bucket = "nome-do-bucket"
    key    = "terraform.tfstate"
    region = "us-east-2"
  }
}
```

---

## EC2 — Instância e configurações (ec2.tf)

### Instância EC2

```hcl
resource "aws_instance" "website-server" {
  ami                    = "ami-0a1b6a02658659c2a" # Amazon Linux 2
  instance_type          = "t3.micro"
  key_name               = aws_key_pair.site-prod.key_name
  vpc_security_group_ids = [aws_security_group.website_sg.id]
  iam_instance_profile   = "EC2-ECR-Role"
  user_data              = file("user_data.sh")

  tags = {
    Name        = "website-server"
    provisioned = "Terraform"
  }
}
```

### Key Pair

O key pair registra a **chave pública SSH** na EC2, permitindo que conexões autenticadas com a chave privada correspondente sejam aceitas.

```hcl
resource "aws_key_pair" "site-prod" {
  key_name   = "site-prod-key"
  public_key = file("~/.ssh/site-prod-key.pub")
}
```

> A chave pública fica na EC2. A chave privada fica com quem vai conectar — no caso deste projeto, armazenada como secret no GitHub Actions.

---

## Security Group — Controle de acesso de rede

O Security Group funciona como um **firewall virtual** da instância EC2, controlando quais conexões são permitidas de entrar (ingress) e sair (egress).

### Por que usar `aws_vpc_security_group_ingress_rule`?

Este projeto usa o recurso separado `aws_vpc_security_group_ingress_rule` em vez de blocos `ingress` dentro do `aws_security_group`. Essa é a abordagem mais atual recomendada pela AWS e pela HashiCorp, pois evita conflitos quando regras são gerenciadas de forma independente.

### Regras de entrada (ingress)

```hcl
# SSH — acesso remoto à instância
resource "aws_vpc_security_group_ingress_rule" "allow_ssh" {
  security_group_id = aws_security_group.website_sg.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 22
  ip_protocol       = "tcp"
  to_port           = 22
}

# HTTP — acesso ao site na porta 80
resource "aws_vpc_security_group_ingress_rule" "allow_http" {
  security_group_id = aws_security_group.website_sg.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 80
  ip_protocol       = "tcp"
  to_port           = 80
}

# HTTPS — acesso seguro ao site na porta 443
resource "aws_vpc_security_group_ingress_rule" "allow_https" {
  security_group_id = aws_security_group.website_sg.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 443
  ip_protocol       = "tcp"
  to_port           = 443
}
```

### Regra de saída (egress)

```hcl
# Permite todo tráfego de saída
resource "aws_vpc_security_group_egress_rule" "allow_all_outbound" {
  security_group_id = aws_security_group.website_sg.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = -1
}
```

> `ip_protocol = -1` significa **todos os protocolos**, liberando qualquer tipo de tráfego de saída. Isso é necessário para que a EC2 consiga acessar a internet — por exemplo, para fazer `docker pull` do ECR.

### Resumo das portas abertas

| Porta | Protocolo | Motivo |
|---|---|---|
| 22 | TCP | SSH — conexão remota e deploy via GitHub Actions |
| 80 | TCP | HTTP — acesso ao site pelos usuários |
| 443 | TCP | HTTPS — acesso seguro (preparado para SSL futuro) |
| Saída | Todos | Permitir que a EC2 acesse a internet |

---

## user_data.sh — Inicialização automática da EC2

O `user_data` é um script que a AWS executa **automaticamente na primeira vez que a instância é iniciada**. Ele é ideal para instalar dependências e configurar o ambiente sem precisar acessar a máquina manualmente.

```bash
#!/bin/bash
sudo su
yum update -y
yum install -y docker
service docker start
usermod -a -G docker ec2-user
```

### O que cada linha faz

| Linha | Descrição |
|---|---|
| `#!/bin/bash` | Define que o script deve ser executado pelo bash |
| `sudo su` | Eleva para root para garantir permissão em todos os comandos |
| `yum update -y` | Atualiza todos os pacotes do sistema operacional |
| `yum install -y docker` | Instala o Docker na instância |
| `service docker start` | Inicia o serviço do Docker |
| `usermod -a -G docker ec2-user` | Adiciona o usuário `ec2-user` ao grupo `docker`, permitindo rodar comandos Docker sem precisar de `sudo` |

### Por que o user_data é importante?

Sem o `user_data`, a EC2 subiria sem o Docker instalado. O pipeline de deploy tentaria executar `docker pull` e `docker run` e falharia porque o Docker não existiria na máquina. O `user_data` garante que a instância já esteja preparada para receber a aplicação assim que for criada pelo Terraform.

> ⚠️ O `user_data` só é executado **uma vez**, na primeira inicialização. Se precisar rodar novamente, é necessário recriar a instância.

---

## Boas práticas aprendidas

- Usar `aws_vpc_security_group_ingress_rule` separado em vez de blocos `ingress` dentro do security group
- Nunca commitar arquivos `.tfstate`, `.tfvars` ou chaves SSH no repositório
- Usar backend remoto (S3) para armazenar o estado do Terraform
- Sempre rodar `terraform plan` antes do `apply` para revisar mudanças
- Usar `force_delete = true` em recursos que podem ter conteúdo no momento do destroy
- Usar `user_data` para automatizar a configuração inicial da EC2

---

## .gitignore recomendado para projetos Terraform

```
*.tfstate
*.tfstate.backup
*.tfvars
.terraform/
*.pem
*.pub
site-prod-key
```

[Go to Previous](arquitetura.md)  | [Go to Next](cicd.md)
