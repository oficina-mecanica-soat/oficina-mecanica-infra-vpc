# 🌐 Oficina Mecânica Infra VPC

Infraestrutura de rede AWS da **Oficina Mecânica**, criada para a **Fase 3 do Tech Challenge FIAP**.

Este repositório provisiona somente a base de rede compartilhada pelos demais recursos da solução. A ideia é ser simples, previsível e fácil de operar: VPC primeiro, depois RDS, Kubernetes, Lambda/API Gateway e aplicação.

---

## 🎯 Objetivo

Criar a fundação de rede do ambiente `development` na AWS, mantendo a separação por responsabilidade definida para a Fase 3.

Este repo cuida de:

- VPC da solução.
- Subnets públicas e privadas em duas zonas de disponibilidade.
- Internet Gateway para entrada/saída pública controlada.
- NAT Gateway para saída das subnets privadas.
- Route tables públicas e privadas.
- Publicação dos outputs principais no AWS Systems Manager Parameter Store.

---

## 🧭 Papel na arquitetura

```text
Internet
   │
   ▼
API Gateway ───────────────┐
                            │
                         VPC Link
                            │
                            ▼
                  NLB interno da API
                            │
                            ▼
              EKS / oficina-mecanica-api
                            │
                            ▼
                    RDS SQL Server
```

A VPC é a primeira peça da infraestrutura. Os demais repositórios devem consumir seus identificadores via **SSM Parameter Store**, evitando cópia manual de IDs entre pipelines.

---

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

## 📁 Estrutura

```text
.
├── .github/workflows/                    # CI/CD do Terraform
└── infra/terraform/
    ├── environments/dev/                 # Ambiente development
    └── modules/vpc/                      # Módulo reutilizável de VPC
```

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

Para aplicar localmente:

```powershell
terraform -chdir="infra/terraform/environments/dev" apply
```

Para destruir localmente:

```powershell
terraform -chdir="infra/terraform/environments/dev" destroy
```

> Preferência do projeto: usar a esteira do GitHub Actions para `apply` e `destroy`, mantendo rastreabilidade.

---

## 🔁 CI/CD

A esteira segue o mesmo modelo da API, mas focada somente em Terraform:

| Workflow | Quando roda | O que faz |
| --- | --- | --- |
| `🧪 CI Development` | Pull request para `develop` | Verifica formatação, inicialização e validação do Terraform. |
| `🔎 CI Release` | Pull request para `release` ou `release/**` | Verifica formatação, inicialização e validação do Terraform. |
| `🛡️ CI Production` | Pull request para `main` | Verifica formatação, inicialização e validação do Terraform. |
| `🚀 CD Development` | Push na `develop` | Executa `apply` ou `destroy` em `development`. |
| `☁️ AWS Deploy` | Chamado pelo CD de desenvolvimento | Executa `apply` ou `destroy` da VPC conforme controle versionado. |
| `🔀 CD Release` | Push na `release` ou `release/**` | Registra promoção lógica para homologation e abre PR para `main` quando habilitado. |
| `🏁 CD Production` | Push na `main` | Registra promoção lógica para production. |

A ação real do Terraform é controlada por:

```text
infra/terraform/environments/dev/terraform-action.env
```

Valores aceitos:

```env
TERRAFORM_ACTION=apply
TERRAFORM_ACTION=destroy
```

### Proteção de branches

As branches `develop`, `release`, `release/*` e `main` usam dois rulesets ativos:

- **🔒 Proteção Git Flow, sem bypass:** PR obrigatório, conversas resolvidas, checks `🔀 01 · Validar fluxo de branches` e `🚦 03 · Quality gate` aprovados e bloqueio de push direto, force push e deleção. A lista de bypass fica vazia, inclusive para admins e maintainers.
- **👥 Aprovação de PR:** uma aprovação humana, descarte de aprovações antigas e aprovação por alguém diferente do último autor do push. Somente esta regra permite bypass via PR para `geoscabio`, `sousagabriel14`, maintainers e admins.

O bypass dispensa a revisão de outra pessoa, nunca o fluxo ou os checks. O GitHub permite abrir um PR fora do caminho `branch de trabalho -> develop -> release -> main`, mas a validação bloqueia seu merge. Hotfix permanece pós-entrega, sem exceção habilitada.

O ruleset de fluxo usa `strict_required_status_checks_policy=false` para não exigir promoção inversa entre branches; checks continuam obrigatórios e conflitos reais precisam ser resolvidos. O proprietário ainda pode alterar as configurações administrativas: o bloqueio depende dos rulesets ativos e da integridade dos workflows de validação.

Repetir este padrão em cada novo repositório. A configuração detalhada está no [guia de GitHub Actions da API](https://github.com/geoscabio/oficina-mecanica-api/blob/develop/docs/deploy/github-actions.md#-proteções-obrigatórias-recomendadas).

O título de execução do `🔀 CD Release` é `🔀 Registrar deploy em release`; produção só é registrada depois do merge em `main`.

---

## 🧨 Ordem de operação

### Apply da solução completa

```text
1. oficina-mecanica-infra-vpc
2. oficina-mecanica-infra-rds e oficina-mecanica-infra-kubernetes
3. oficina-mecanica-auth-lambda
4. oficina-mecanica-api
5. oficina-mecanica-infra-api-gateway
```

### Destroy da solução completa

```text
1. oficina-mecanica-infra-api-gateway
2. oficina-mecanica-api
3. oficina-mecanica-auth-lambda
4. oficina-mecanica-infra-kubernetes e oficina-mecanica-infra-rds
5. oficina-mecanica-infra-vpc
```

A VPC deve ser a última a ser destruída, porque os demais recursos dependem dela.

---

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

## ✅ Boas práticas adotadas

- Responsabilidade única por repositório e por pipeline.
- Terraform separado por `environment` e `module`.
- Tags padronizadas em todos os recursos.
- Outputs compartilhados por SSM Parameter Store.
- Controle explícito de `apply` e `destroy`.
- Validação real da VPC após `apply` e após `destroy`.
- Nomenclatura com prefixo `oficina-mecanica-*`.

Documentação central e arquitetura completa: [README da Oficina Mecânica API](https://github.com/geoscabio/oficina-mecanica-api).

### Reconciliação de destroy após reset do laboratório

O destroy restaura o cache Terraform existente e usa uma configuração temporária sem data sources de contratos SSM. O state original não é editado nem descartado: o provider confirma os recursos ausentes durante o plano de destruição. Somente o plano salvo, sem criação ou atualização, pode ser aplicado. Ao terminar, a esteira verifica que o state não contém recursos gerenciados antes de persistir o cache final. Falhas de autenticação, permissão ou rede bloqueiam a operação.

Na API, vínculos Kubernetes só podem ser retirados por blocos declarativos removed com destroy=false após ResourceNotFoundException confirmar que o EKS registrado no state não existe. Com EKS existente, permanece a configuração normal de destroy. Não há uso de state rm, import ou mv.

O retorno de terraform-action.env para apply é reconhecido como mudança de infraestrutura pelo detector de CD. O bloqueio `AWS_DEPLOY_ENABLED` descrito abaixo separa essa preparação da execução real; somente autorize nova subida em janela própria.

#### State isolado por conta AWS

As chaves de cache incluem o ID da conta retornado pela autenticação AWS, antes do ambiente e da execução. A restauração só pesquisa esse escopo, sem fallback para chaves históricas sem conta. Os caches anteriores permanecem preservados como histórico do laboratório encerrado 058264466885, inclusive quando contêm referências que a conta atual não consegue consultar. AccessDenied não é convertido em ausência.

Sem cache no escopo atual, a esteira bloqueia apply. Um destroy dedicado só pode iniciar state novo depois de consultas read-only confirmarem que não existem recursos ou contratos do projeto na conta atual. Recursos internos do laboratório são preservados. Depois disso, o plano exclusivo de destroy e a validação de state vazio produzem o cache usado pelas próximas execuções da mesma conta. Em laboratório com recursos existentes e cache ausente, é necessário recuperar o state correto; o bootstrap não libera provisionamento automático.

### Janela explícita de execução AWS — F4-04

Em 11/11/2026, a preparação do retorno a `apply` acrescenta à VPC o bloqueio já usado na API, no Execução e em Kubernetes: `AWS_DEPLOY_ENABLED=true` é obrigatório no CD Development e no workflow reutilizável. Ausente/false/valor inválido bloqueia AWS. O reutilizável também exige branch develop. A variable foi cadastrada como `false` no repositório; não configurar um override divergente no Environment.

Assim, o PR pode preparar `TERRAFORM_ACTION=apply` sem provisionar no merge. A mudança de variable não dispara execução por si só; a janela autorizada deverá usar o fluxo existente no commit correto. Aprovar/mesclar código não autoriza habilitar AWS. Promoção de mudança implantável bloqueada continua sem avançar como se houvesse deploy bem-sucedido.

O futuro apply usará o state/cache isolado pela conta e o plano salvo já existentes. Não cria plano alternativo, altera topologia/CIDRs/nomes, descarta caches ou modifica os guardrails de destroy. Prevê somente os recursos de rede já descritos neste README: VPC, subnets, IGW/NAT, EIP, rotas e contratos SSM. EKS/RDS/serviços continuam em seus owners e etapas seguintes.

A validação do bloqueio executa a etapa real de resolução em fixture temporária no CI, sem AWS: controle ausente/false/inválido ou branch diferente de develop não geram saída; true em develop com apply apenas resolve a ação. O teste está no YAML existente, sem novos arquivos/pastas de scripts. Nenhuma implantação foi executada para validar esse controle.

## Promoção cumulativa — AUD-003

A abertura/manutenção automática do PR `develop` → `release` compara todo o conteúdo implantável pendente, seguindo a regra já aprovada na API/Execução. Uma alteração documental não libera código/infraestrutura anterior sem operação confirmada. Detector e guarda compartilham os mesmos caminhos, preservando o escopo de deploy existente.

A guarda exige sucesso da operação atual ou do CD do último commit implantável na linha principal de develop, incluindo o job de apply/destroy da tentativa atual. Deploy pulado/falho, rerun falho, histórico ausente ou erro de Git/GitHub bloqueiam; documentação após operação comprovada continua permitida. É conservadora quando a evidência exigida não está disponível. Não consulta AWS.

Preserva `AWS_DEPLOY_ENABLED`, `AUTO_PR_ENABLED`, environments, plano salvo, state/cache, guardrails de destroy e merge humano. Não altera ancestralidade (PIPE-005), recursos, contratos SSM ou `terraform-action.env`. PRs existentes continuam sujeitos à revisão humana; a trava protege sua abertura/manutenção pelo CD. Nesta rodada, integrar somente em `develop`, sem promover release/main, sem implantação e com laboratório offline. Testes ficam inline no CI existente, sem novos arquivos ou pastas auxiliares.

## Artefatos por tentativa — AUD-005

Planos Terraform e, na Lambda, o pacote planejado usam `github.run_id` e `github.run_attempt` no nome de publicação/download. Os caches de state também usam conta AWS, ambiente, execução e tentativa. A restauração mantém o prefixo existente de conta/ambiente para recuperar state anterior, sem consultar caches de outra conta ou ambiente; não há fallback equivalente para planos/pacotes. O salvamento final com `always()` permanece para preservar state de apply/destroy parcialmente concluído. Nenhum cache antigo é apagado.

Uma nova tentativa precisa gerar e publicar seus próprios artefatos. Reexecutar somente jobs falhos, mantendo o planejamento bem-sucedido de uma tentativa anterior, não fornece artefatos para a nova tentativa e deve bloquear o download. Para repetir o ciclo, reexecute todos os jobs e gere um novo plano a partir do state recuperado; não renomeie nem copie artefatos antigos nem pule o planejamento. Não é autorizada uma reexecução AWS por este ajuste.

Testes inline verificam nomes de produtor/consumidor, isolamento entre tentativas/execuções/contas/ambientes, ausência de fallback silencioso de artefatos e preservação da configuração de state em falha. São verificações locais/simuladas, não prova de um rerun real de apply no laboratório. Mantém bloqueios AWS, environments, plano salvo, ação versionada, recursos e contratos. Sem novas pastas/arquivos auxiliares. Integrar somente em `develop`, com merge por Gabriel; laboratório offline, sem implantação ou promoção para release/main.
