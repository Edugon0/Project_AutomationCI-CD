# 🚀 DevOps Project — Automação de Infraestrutura AWS
 
![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat&logo=terraform&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat&logo=amazonaws&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=flat&logo=githubactions&logoColor=white)
 
## 📌 Sobre o Projeto
 
Este projeto documenta a construção de uma infraestrutura completa na AWS utilizando práticas modernas de DevOps. O objetivo foi automatizar desde a criação da infraestrutura até o deploy da aplicação, eliminando processos manuais e tornando o fluxo de entrega mais rápido e confiável.
 
O projeto é dividido em **dois repositórios** com pipelines CI/CD independentes:
 
| Repositório | Descrição |
|---|---|
| [Full-automation-with-CI-CD-](https://github.com/Edugon0/Project-automation-Terraform) | Provisionamento de infraestrutura AWS com Terraform |
| [Automation-DeploySite-instance-ec2](https://github.com/Edugon0/Automation-DeploySite-instance-ec2) | Deploy automático da aplicação na EC2 |

Obs.: Esses dois repositórios estão por enquanto privado por questão de segurança, pois ainda tem alguns dados sensivel dentro dos repositorios, assim que possivel firá disponivel
 
---
 
## 🏗️ Arquitetura

![Arquitetura](docs/arquitetura.svg)

---

## 🛠️ Tecnologias Utilizadas
 
- **Terraform** — Infraestrutura como código (IaC)
- **AWS EC2** — Servidor da aplicação
- **AWS ECR** — Repositório de imagens Docker
- **AWS S3** — Armazenamento do estado remoto do Terraform
- **AWS IAM** — Controle de acesso via OIDC
- **Docker** — Containerização da aplicação
- **GitHub Actions** — Pipelines de CI/CD
- **Nginx** — Servidor web dentro do container
---
 
## 📂 Documentação
 
- [🏗️ Arquitetura detalhada](arquitetura.md)
- [🌱 Terraform — o que aprendi](terraform.md)
- [⚙️ Pipelines CI/CD](cicd.md)
- [🐳 Docker + ECR](docker-ecr.md)
- [🔥 Problemas e soluções](problemas-solucoes.md)
---
## 💡 Principais Aprendizados
 
- Provisionamento de infraestrutura AWS com Terraform de forma automatizada
- Configuração de autenticação segura entre GitHub Actions e AWS via OIDC
- Criação de pipelines CI/CD completos com controle de apply/destroy
- Gerenciamento de estado remoto do Terraform com S3
- Deploy automatizado de containers Docker em instâncias EC2
- Diagnóstico e resolução de problemas reais de infraestrutura
