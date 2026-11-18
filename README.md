# 🌐 Oficina Mecânica Infra VPC

Rede compartilhada da solução: VPC, subnets públicas/privadas, rotas, Internet Gateway e NAT. Publica os contratos usados pelas demais esteiras. É a primeira infraestrutura no apply e a última no destroy.

## Dependências

Não depende de outra esteira. Consumidores usam `/oficina-mecanica/development/vpc/*` e `/oficina-mecanica/development/status/vpc`. Após apply, o CD confere recursos/contratos; após destroy, confere a ausência. Não remover enquanto houver consumidores.


## 📦 Recursos criados

| Recurso | Nome padrão | Responsabilidade |
| --- | --- | --- |
| VPC | `oficina-mecanica-vpc-dev` | Isolamento lógico da rede AWS. |
| Public Subnets | `oficina-mecanica-vpc-dev-public-*` | NAT Gateway e recursos públicos controlados. |
| Private Subnets | `oficina-mecanica-vpc-dev-private-*` | EKS, RDS e integrações privadas. |
| Internet Gateway | `oficina-mecanica-vpc-dev-igw` | Saída/entrada pública da VPC. |
| NAT Gateway | `oficina-mecanica-vpc-dev-nat` | Saída para internet das subnets privadas. |
| Route Tables | `oficina-mecanica-vpc-dev-*-rt` | Rotas públicas e privadas. |
| SSM Parameters | `/oficina-mecanica/development/vpc/*` | Compartilhamento dos outputs entre repositórios. |
| SSM Status | `/oficina-mecanica/development/status/vpc` | Marcador usado por repositórios dependentes. |

---

## 🚀 Execução local

Pré-requisitos:

- Terraform `1.15.8` instalado.
- AWS CLI autenticado no AWS Academy.
- Região `us-east-1`.

```powershell
terraform -chdir="infra/terraform/environments/dev" init
terraform fmt -recursive infra/terraform
terraform -chdir="infra/terraform/environments/dev" validate
terraform -chdir="infra/terraform/environments/dev" plan
```

Apply e destroy reais são exclusivos da esteira GitHub Actions, por PR em `develop`, controle versionado e janela autorizada. Os comandos locais acima são somente de validação e planejamento; não aplicar nem destruir recursos localmente.

## 🔐 Secrets e variáveis

Configurar no GitHub Environment `development`:

| Tipo | Nome | Uso |
| --- | --- | --- |
| Secret | `AWS_ACCESS_KEY_ID` | Autenticação AWS. |
| Secret | `AWS_SECRET_ACCESS_KEY` | Autenticação AWS. |
| Secret | `AWS_SESSION_TOKEN` | Sessão temporária do AWS Academy. |
| Variable | `AWS_REGION` | Região AWS, padrão `us-east-1`. |
| Variable | `AUTO_PR_ENABLED` | Abre PR automático para `release` quando `true`. |
| Variable | `RELEASE_BRANCH` | Branch de release, padrão `release`. |

Nenhum segredo deve ser versionado no repositório.

Os itens devem ser criados em **Settings > Environments > development**. `AUTO_PR_ENABLED` e `RELEASE_BRANCH` também são lidos como Variables do repositório pelos workflows de promoção; seus valores são configurações não sensíveis.

Após `apply`, valide a VPC e os parâmetros `/oficina-mecanica/development/vpc/*` e `/oficina-mecanica/development/status/vpc`. Após `destroy`, a esteira confirma a ausência dos recursos e contratos SSM antes de considerar a execução concluída.

---

## Operação e documentação

- [Visão geral, arquitetura e ordem das esteiras](https://github.com/oficina-mecanica-soat/oficina-mecanica-api#deploy-aws).
- [Configuração, primeira aplicação e recuperação após Reset](https://github.com/oficina-mecanica-soat/oficina-mecanica-api/blob/develop/docs/deploy/deploy-aws.md).
- [Collection consolidada e ambientes Local/Development](https://github.com/oficina-mecanica-soat/oficina-mecanica-api/blob/develop/docs/evidencias/postman.md).

Todo provisionamento, atualização e destroy na AWS ocorre pelas esteiras oficiais. O controle é `infra/terraform/environments/dev/terraform-action.env`; destroy exige alteração dedicada e verificação dos consumidores. `AWS_DEPLOY_ENABLED` autoriza a janela real, e `AUTO_PR_ENABLED` controla somente a abertura de promoções. Alterar uma Variable não dispara o CD.

Um ambiente disponível pode ser testado sem destroy. Sem estado compatível, alguns owners exigem primeiro uma recuperação por destroy dedicado com inventário vazio comprovado. Não apague recursos ou caches para contornar o bloqueio; consulte o guia antes de iniciar uma conta ou retomar um laboratório.

Fluxo Git: branch de trabalho → PR para `develop` → `release` → `main`. O CD físico ocorre em development; release e production registram promoções lógicas. PRs e merges permanecem sujeitos aos checks e à revisão. Mudanças somente em README/docs não exigem deploy físico pelos detectores atuais.
