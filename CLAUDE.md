# Bridgge — instruções para agentes

Contexto operacional dos projetos da Bridgge. Leia antes de mexer em qualquer repo daqui.

## Regra principal: nada vai a produção sem verificação

**`main` = produção.** Todo push na `main` dos repos do Clover dispara build + deploy
automático. Não existe staging, nem testes no pipeline, nem aprovação manual.
Um merge errado chega ao cliente em ~2 minutos.

Por isso: **trabalhe sempre em branch + PR.** Nunca commite direto na `main`.

```bash
git checkout -b fix/descricao-curta
# ... edita, testa ...
git push -u origin fix/descricao-curta
gh pr create --fill
```

O merge é a publicação. É decisão humana — um agente não faz merge sem pedido explícito.

## Checklist de integridade antes de publicar

Rodar **todos** os itens aplicáveis. Se algum falhar, não publique.

### 1. O código carrega
```bash
python -c "import ast; ast.parse(open('main.py',encoding='utf-8').read())"   # Python
npm run build                                                                # Node/Next
```

### 2. Teste de comportamento, não só de sintaxe
Exercite o caminho que você mudou **e** o caminho que já funcionava.
Sintaxe válida não prova que a lógica está certa.

Para APIs FastAPI, dá para testar sem banco real — substitua `execute_query`
por um espião e use `TestClient`:

```python
import psycopg2.pool
class FakePool:
    def __init__(self, *a, **kw): self.maxconn = kw.get("maxconn")
psycopg2.pool.ThreadedConnectionPool = FakePool   # antes de importar main
```

Verifique que a entrada inválida é **bloqueada antes de tocar o banco**,
e que a entrada válida **continua funcionando** (não-regressão).

### 3. Cubra todas as portas de entrada
Validar só o modelo Pydantic **não basta**. Um mesmo campo entra por vários caminhos:

- corpo de `POST`/`PUT` → modelo Pydantic
- query param (`?data=...`) → precisa de `Query(pattern=...)`
- `PATCH` que recebe `dict` cru → precisa de validação explícita no handler

Erro real já cometido aqui: validar só o `POST` e deixar `GET`, `PATCH` passando direto.

### 4. Mensagem de commit sem metacaracteres de shell
**Não use crases, `$(...)`, `${...}` em mensagens de commit.**
O workflow de deploy passa a mensagem para o shell. Isso já derrubou um deploy
(exit 127). Foi corrigido usando `env:`, mas o hábito continua valendo.

### 5. Depois do merge, confirme que chegou
Deploy "verde" no GitHub não significa que subiu. Já aconteceu de a imagem ser
publicada e o servidor nunca ser avisado.

```bash
gh run watch <run-id> -R Bridgge-IA/<repo>
ssh root@195.200.6.174 "docker ps --format '{{.Names}} {{.Image}} {{.Status}}'"
curl -s -o /dev/null -w "%{http_code}\n" https://clover.bridgge.com.br
```

Confirme que o **container foi recriado** (data recente), não só que o workflow passou.

## Infraestrutura

**VPS** Hostinger — `195.200.6.174` (`srv929764`), Ubuntu 24.04, **1 vCPU / 3,8 GB / 50 GB**.
Backup diário automático pela Hostinger (4 snapshots).

**Coolify 4.3.17** roda nessa VPS e orquestra os containers.
Painel: `https://coolify.bridgge.com.br` (prefira o domínio ao `http://IP:8000`, que é sem TLS).

Máquina pequena: 1 vCPU e pouca RAM, **sem swap**. Antes de subir serviço novo,
confira `free -h`.

### Fluxo de deploy

```
push na main → GitHub Actions → build Docker → ghcr.io
                              → POST webhook → Coolify → container novo
```

O trabalho está centralizado em `ci-workflows/.github/workflows/build-deploy-coolify.yml`.
Os 4 repos do Clover só o chamam. Mudou o processo de deploy? Mexe num arquivo só.

### Serviços em produção

| Domínio | Repo |
|---|---|
| `clover.bridgge.com.br` | `clover-front` |
| `clover-api.bridgge.com.br` | `clover-api` |
| `bridgge.com.br` | `bridgge-front` |
| `auth.bridgge.com.br` | `bridgge-auth` |

`clover-ia` e `clover-webhook` têm repo e imagem, mas os containers estão **parados**
(bot de WhatsApp fora de uso — clientes reservam pelo site).

### Secrets por repo

| Secret | O que é |
|---|---|
| `COOLIFY_TOKEN` | token de API do Coolify, permissão `Deploy` |
| `COOLIFY_WEBHOOK_URL` | `https://coolify.bridgge.com.br/api/v1/deploy?uuid=<UUID-DO-APP>` |

O `?uuid=` é obrigatório — o workflow anexa `&message=...` no final.
URL sem query string quebra com `curl_error/000` (falha em milissegundos, não é timeout).

Variáveis de ambiente da aplicação ficam **no painel do Coolify**, não no Git.
`.env.example` é só modelo. Se o painel mostrar "Changes pending", as variáveis
ainda não foram aplicadas ao container — precisa de Redeploy.

## Diagnóstico

Quando um deploy falhar, o código de erro diz onde olhar:

| Sintoma | Causa provável |
|---|---|
| `exit 127` | metacaractere de shell na mensagem de commit |
| `HTTP 401` | token do Coolify ausente, expirado ou revogado |
| `HTTP 405` | endpoint mudou de método (o Coolify migrou GET → POST) |
| `curl_error/000` em ms | URL de webhook malformada (sem `?uuid=`) |
| `pool exhausted` | picos de concorrência — `maxconn` vs `--workers` |

Os logs do servidor são a fonte de verdade:

```bash
ssh root@195.200.6.174 "docker logs --tail 50 --timestamps <container>"
```

**Erros constantes por dia ≠ vazamento.** Vazamento cresce até travar e não se recupera.
Erro em rajada com recuperação automática é concorrência.

## Cuidados

- **Nunca cole senha ou token no chat.** Use `gh secret set` no seu terminal.
  Se um segredo vazar numa conversa, troque-o.
- **Banco Postgres fechado para fora** (porta 5434). Para testar local, use túnel SSH —
  mas lembre que é o banco de **produção**: `GET` é seguro, `POST`/`PATCH` alteram dados reais.
- **Antes de parar container**, confira se algum serviço depende dele. Prefira
  `docker stop` a remover: preserva volumes e é reversível com `docker start`.
- **Ações que afetam clientes** (merge que publica, parar serviço no ar, arquivar repos)
  são decisão humana. Um agente pergunta antes.

## Ambiente local

`C:\git\Bridgge\Bridgge-IA\` contém os 7 repos ativos. Outros clientes ficam em
`C:\git\` e não devem ser tocados em tarefas da Bridgge.

Repos obsoletos (HubWhatsServices, Casa do Capitão) foram removidos do disco mas
**continuam no GitHub**. Ver `DIAGNOSTICO-ORGS.md` (mantido localmente em `C:gitBridgge`).
