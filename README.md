# n8n — producao + teste (Coolify)

Instancia do n8n rodando via Docker Compose no Coolify, com dois ambientes
independentes (producao e teste) apontando para o mesmo servidor Postgres,
cada um no seu proprio banco. Arquivos binarios dos workflows (ex: PDFs
gerados, planilhas processadas) ficam salvos em disco local (volume Docker),
nao dentro do Postgres.

## Arquivos deste projeto

- `compose.yaml` — a definicao do servico n8n. E o MESMO arquivo usado para
  os dois ambientes; o que muda entre producao e teste sao as variaveis de
  ambiente e o dominio configurados no Coolify.
- `.env.example` — lista de variaveis que precisam existir. So documentacao;
  os valores reais entram direto no Coolify (nunca no git).
- `.env` — opcional, so se voce quiser rodar `docker compose up` localmente
  no WSL pra testar antes de subir no Coolify. Fica fora do git.

A pasta `data/` que existia foi removida: bind mount local (`./data:...`)
nao sobrevive com garantia a redeploys no Coolify (ele pode reclonar o
repositorio a cada deploy). O `compose.yaml` usa um **named volume**
(`n8n_data`), que e a forma que o Coolify gerencia como storage persistente
de verdade.

## 1. Preparar o Postgres (host separado)

Quando o Postgres estiver no ar, rode uma vez (via psql, com um usuario
admin) para criar os dois bancos e um usuario dedicado por ambiente:

```sql
CREATE DATABASE n8n_producao;
CREATE DATABASE n8n_teste;

CREATE USER n8n_prod WITH PASSWORD 'defina_uma_senha_forte';
CREATE USER n8n_test WITH PASSWORD 'defina_outra_senha_forte';

GRANT ALL PRIVILEGES ON DATABASE n8n_producao TO n8n_prod;
GRANT ALL PRIVILEGES ON DATABASE n8n_teste TO n8n_test;

-- Postgres 15+: o dono do schema public precisa conceder isso explicitamente
\c n8n_producao
GRANT ALL ON SCHEMA public TO n8n_prod;
\c n8n_teste
GRANT ALL ON SCHEMA public TO n8n_test;
```

Usar um usuario por ambiente (em vez de um so pra tudo) evita que um bug ou
uma automacao de teste consiga mexer no banco de producao.

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
   | `DB_POSTGRESDB_HOST` | igual nos dois (mesmo host Postgres) | |
   | `DB_POSTGRESDB_DATABASE` | `n8n_producao` | `n8n_teste` |
   | `DB_POSTGRESDB_USER` | `n8n_prod` | `n8n_test` |
   | `DB_POSTGRESDB_PASSWORD` | senha do n8n_prod | senha do n8n_test |

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
# preencha .env com valores de teste (pode ser um Postgres local ou o mesmo remoto)
docker compose up
```

Acesse `http://localhost:5678`.

## Duvidas comuns

- **"Preciso mexer na pasta `data/`?"** Nao, foi removida — o volume
  `n8n_data` e criado e gerenciado pelo proprio Docker/Coolify.
- **Os dois ambientes competem pelo mesmo volume?** Nao, o nome do volume
  inclui `N8N_ENV_NAME`, entao `n8n_data_producao` e `n8n_data_teste` sao
  volumes distintos mesmo usando o mesmo `compose.yaml`.
- **Da pra promover um workflow de teste pra producao?** Sim, no editor do
  n8n existe export/import de workflow (JSON), ou uso do recurso de
  versionamento por Git do n8n (Enterprise) se quiser algo mais formal.
