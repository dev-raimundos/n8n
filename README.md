# n8n — producao + teste (Coolify)

Instancia do n8n rodando via Docker Compose no Coolify, com dois ambientes
independentes (producao e teste), cada um com seu proprio Postgres embutido
no mesmo compose — nenhum dos dois depende de um servidor de banco externo.
Arquivos binarios dos workflows (ex: PDFs gerados, planilhas processadas)
ficam salvos em disco local (volume Docker), nao dentro do Postgres.

## Arquivos deste projeto

- `compose.yaml` — define os servicos `postgres` e `n8n`. E o MESMO arquivo
  usado para os dois ambientes; o que muda entre producao e teste sao as
  variaveis de ambiente e o dominio configurados no Coolify.
- `.env.example` — lista de variaveis que precisam existir. So documentacao;
  os valores reais entram direto no Coolify (nunca no git).
- `.env` — opcional, so se voce quiser rodar `docker compose up` localmente
  no WSL pra testar antes de subir no Coolify. Fica fora do git.

A pasta `data/` que existia foi removida: bind mount local (`./data:...`)
nao sobrevive com garantia a redeploys no Coolify (ele pode reclonar o
repositorio a cada deploy). O `compose.yaml` usa **named volumes**
(`n8n_data` e `n8n_postgres_data`), que e a forma que o Coolify gerencia como
storage persistente de verdade.

## 1. Postgres embutido (sem host separado)

O servico `postgres` sobe junto no mesmo compose e cria o banco/usuario
sozinho na primeira subida (via as variaveis `POSTGRES_DB`/`POSTGRES_USER`/
`POSTGRES_PASSWORD`, preenchidas a partir de `DB_POSTGRESDB_*`). O n8n fala
com ele pelo nome do servico (`DB_POSTGRESDB_HOST=postgres`) na rede interna
do Docker — sem SSL, sem IP/porta pra configurar, sem depender de nenhum
outro servidor estar de pe.

Cada ambiente (producao/teste) tem seu proprio volume de dados do Postgres
(`n8n_postgres_data_producao` / `n8n_postgres_data_teste`), entao os dois
bancos ficam completamente isolados mesmo compartilhando o mesmo
`compose.yaml`.

## 2. Estrutura no Coolify

O Coolify organiza tudo como `Servidor -> Projeto -> Ambiente -> Recurso`.
Isso encaixa exatamente com o que voce quer:

1. Crie um **Projeto** (ex: `n8n-automacoes`).
2. Dentro dele, use os dois **Ambientes** que o Coolify ja oferece:
   `production` e `staging` (renomeie `staging` para `teste` se quiser).
3. Em cada ambiente, crie um **Recurso do tipo Docker Compose** apontando
   pra este repositorio (ou cole o conteudo de `compose.yaml` direto, se
   preferir nao usar git).
4. Configure as variaveis de ambiente do recurso (uma lista por ambiente):

   | Variavel | Producao | Teste |
   |---|---|---|
   | `N8N_ENV_NAME` | `producao` | `teste` |
   | `N8N_ENCRYPTION_KEY` | gerar com `openssl rand -base64 32` | gerar outra, diferente |
   | `DB_POSTGRESDB_DATABASE` | `n8n_producao` | `n8n_teste` |
   | `DB_POSTGRESDB_USER` | `n8n_prod` | `n8n_test` |
   | `DB_POSTGRESDB_PASSWORD` | senha forte, uma por ambiente | senha forte, diferente da de producao |

   Essas tres ultimas variaveis nao apontam pra um banco que ja existe — elas
   definem o banco/usuario que o container `postgres` embutido cria sozinho
   na primeira subida daquele ambiente.

   As duas chaves de criptografia precisam ser **diferentes e permanentes**:
   se voce regenerar a chave depois, todas as credenciais salvas naquele
   ambiente (ex: token de API, senha de SMTP) ficam impossiveis de usar de
   novo, e voce teria que recadastrar tudo.

5. Em cada recurso, na aba do servico `n8n`, defina o **Dominio**. Como o
   n8n escuta na porta 5678 (nao 80), informe a porta junto:
   - Producao: `https://n8n.suaempresa.com.br:5678`
   - Teste: `https://n8n-teste.suaempresa.com.br:5678`

   O Coolify cuida do Traefik + certificado SSL (Let's Encrypt) sozinho a
   partir disso, e preenche `SERVICE_FQDN_N8N` / `SERVICE_URL_N8N`
   automaticamente — que e o que o `compose.yaml` usa para `N8N_HOST` e
   `WEBHOOK_URL`. Voce nao precisa preencher essas duas nas variaveis de
   ambiente manualmente.

6. Deploy. Repita o passo 3-5 no outro ambiente.

## 3. Testando localmente antes de subir (opcional)

```bash
cp .env.example .env
# preencha .env com valores de teste
docker compose up
```

O Postgres sobe junto (servico `postgres` no mesmo compose), entao isso
funciona isolado, sem precisar de nenhum banco externo rodando.

Acesse `http://localhost:5678`.

## Duvidas comuns

- **"Preciso mexer na pasta `data/`?"** Nao, foi removida — o volume
  `n8n_data` e criado e gerenciado pelo proprio Docker/Coolify.
- **Os dois ambientes competem pelo mesmo volume?** Nao, o nome do volume
  inclui `N8N_ENV_NAME`, entao `n8n_data_producao`/`n8n_data_teste` e
  `n8n_postgres_data_producao`/`n8n_postgres_data_teste` sao volumes
  distintos mesmo usando o mesmo `compose.yaml`.
- **E o Postgres compartilhado que era usado antes?** Nao e mais necessario
  pra esse projeto; cada ambiente agora tem seu proprio Postgres embutido no
  compose, isolado dos demais.
- **Da pra promover um workflow de teste pra producao?** Sim, no editor do
  n8n existe export/import de workflow (JSON), ou uso do recurso de
  versionamento por Git do n8n (Enterprise) se quiser algo mais formal.
