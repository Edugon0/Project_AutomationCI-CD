# 🔥 Problemas Encontrados e Soluções
 
Documentação dos principais erros encontrados durante o desenvolvimento do projeto e como foram resolvidos. Essa seção representa o aprendizado prático mais valioso do projeto.
 
---
 
## 1. Token GitHub sem permissão de workflow
 
**Erro:**
```
refusing to allow a Personal Access Token to create or update workflow
`.github/workflows/terraform.yaml` without `workflow` scope
```
 
**Causa:** O Personal Access Token (PAT) criado no GitHub não tinha o escopo `workflow` habilitado.
 
**Solução:** Editar o token em *Settings → Developer settings → Personal access tokens* e marcar a permissão `workflow`.
 
---
 
## 2. Remote origin já existente
 
**Erro:**
```
error: remote origin already exists.
```
 
**Causa:** Tentar adicionar um remote `origin` em um repositório que já tinha um configurado.
 
**Solução:**
```bash
git remote set-url origin https://github.com/Edugon0/nome-do-repo.git
```
 
---
 
## 3. Autenticação GitHub com senha
 
**Erro:**
```
remote: Invalid username or token. Password authentication is not supported.
```
 
**Causa:** O GitHub removeu o suporte a autenticação por senha. É necessário usar Personal Access Token (PAT).
 
**Solução:** Usar o PAT no lugar da senha e salvar as credenciais:
```bash
git config --global credential.helper store
```
 
---
 
## 4. OIDC não autorizado — nome de repositório errado no secret
 
**Erro:**
```
Error: Could not assume role with OIDC: Not authorized to perform
sts:AssumeRoleWithWebIdentity
```
 
**Causa:** O nome do GitHub Secret estava diferente do nome referenciado no arquivo `.yaml` do workflow.
 
**Solução:** Verificar em *Settings → Secrets and variables → Actions* que o nome do secret é **idêntico** ao usado no workflow, incluindo maiúsculas e underscores.
 
---
 
## 5. Repositório dentro de outro repositório
 
**Aviso:**
```
hint: You've added another git repository inside your current repository.
```
 
**Causa:** A pasta `My-Portfolio` tinha seu próprio `.git`, criando um repositório aninhado.
 
**Solução:**
```bash
# Remover o .git interno
rm -rf My-Portfolio/.git
 
# Ou remover do índice sem apagar os arquivos
git rm --cached My-Portfolio
```
 
---
 
## 6. Push rejeitado — remote com commits mais recentes
 
**Erro:**
```
! [rejected] main -> main (fetch first)
Updates were rejected because the remote contains work that you do not have locally.
```
 
**Causa:** O repositório remoto tinha commits que não existiam localmente.
 
**Solução:**
```bash
git pull origin main
git push
```
 
---
 
## 7. SSH Connection timed out
 
**Erro:**
```
ssh: connect to host *** port 22: Connection timed out
```
 
**Causa:** A instância EC2 não tinha IP público associado. Sem `associate_public_ip_address = true` no Terraform, a instância ficou sem IP público e o SSH não conseguia alcançá-la.
 
**Solução:** Adicionar no recurso `aws_instance`:
```hcl
associate_public_ip_address = true
```
 
---
 
## 8. Identity file not accessible — erro no nome da variável SSH
 
**Erro:**
```
Warning: Identity file -prod-key not accessible: No such file or directory.
```
 
**Causa:** O `$` na variável `$site-prod-key` estava sendo interpretado incorretamente pelo shell, quebrando o nome do arquivo.
 
**Solução:** Remover o `$` do caminho do arquivo:
```bash
# ❌ Errado
ssh -i $site-prod-key ...
 
# ✅ Correto
ssh -i site-prod-key ...
```
 
---
 
## 9. Usando chave pública em vez de privada no SSH
 
**Causa:** O workflow estava salvando `TF_PUBLIC_KEY` para conectar via SSH, mas o SSH requer a **chave privada**.
 
**Solução:**
```bash
# ❌ Errado — chave pública não conecta
echo "${{ secrets.TF_PUBLIC_KEY }}" > site-prod-key
 
# ✅ Correto — chave privada para conectar
echo "${{ secrets.TF_PRIVATE_KEY }}" > site-prod-key
chmod 400 site-prod-key
```
 
---
 
## 10. ECR não deletado no terraform destroy
 
**Erro:**
```
RepositoryNotEmptyException: The repository cannot be deleted because
it still contains images
```
 
**Causa:** O Terraform não deleta repositórios ECR que contêm imagens por padrão.
 
**Solução:** Adicionar `force_delete = true` no recurso:
```hcl
resource "aws_ecr_repository" "ecr-site" {
  name         = "site_prod"
  force_delete = true
}
```
 
Ou limpar as imagens antes do destroy no pipeline:
```yaml
- name: Limpar imagens do ECR
  run: |
    aws ecr batch-delete-image \
      --repository-name site_prod \
      --image-ids "$(aws ecr list-images \
        --repository-name site_prod \
        --query 'imageIds[*]' \
        --output json)" \
      --region us-east-2
```


[Go to Next](docker-ecr.md)|[Home to page](README.md)
