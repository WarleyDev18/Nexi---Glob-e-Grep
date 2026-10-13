# Glob e Grep na prática — 7 exemplos num repo fictício da Nexi

Todos os exemplos usam um repositório **fictício** chamado `nexi-cobranca` (serviço de cobrança e notificações). Nenhum dado real da empresa aparece aqui.

## Estrutura do repo fictício

```
nexi-cobranca/
├── config/
│   ├── default.yaml
│   ├── production.yaml
│   └── staging.yaml
├── src/
│   ├── index.ts
│   ├── env.ts
│   ├── api/
│   │   ├── router.ts
│   │   └── handlers/
│   │       ├── boleto.ts
│   │       └── webhook.ts
│   ├── services/
│   │   ├── billing.ts
│   │   ├── mailer.ts
│   │   └── retry/
│   │       └── backoff.ts
│   └── utils/
│       └── format.ts
├── tests/
│   ├── billing.test.ts
│   ├── api/
│   │   └── webhook.test.ts
│   └── services/
│       └── mailer.test.ts
├── migrations/
│   ├── 001_create_faturas.sql
│   └── 002_add_status_pagamento.sql
└── package.json
```

> **Glob** busca por **nome/caminho de arquivo**. **Grep** busca por **conteúdo dentro** dos arquivos. Regra prática: não sei *onde* está o arquivo → Glob; sei *o que* procuro dentro dele → Grep.

---

## Exemplo 1 — `*` vs `**` (a diferença de escopo)

**Situação:** quero listar os arquivos TypeScript da camada `src/services/`. Vou mostrar os dois patterns lado a lado.

### 1A — `*` (não cruza pasta)

**Pattern:** `src/services/*.ts`

**Resultado esperado:**
```
src/services/billing.ts
src/services/mailer.ts
```
`src/services/retry/backoff.ts` **não** aparece: ele está numa subpasta, e o `*` casa apenas dentro de um único nível de diretório (não casa `/`).

### 1B — `**` (cruza pastas)

**Pattern:** `src/services/**/*.ts`

**Resultado esperado:**
```
src/services/billing.ts
src/services/mailer.ts
src/services/retry/backoff.ts
```
Agora `backoff.ts` entra, porque `**` casa zero ou mais níveis de diretório.

**Por que um e não o outro:** a escolha depende da pergunta. Se quero só os serviços "de primeiro nível" (os que o `index.ts` importa direto), `*` é o certo: ele me dá uma resposta mais curta e evita ruído de helpers internos. Se quero *todo* o código do módulo (por exemplo, para checar se algum arquivo usa uma API depreciada), `**` é obrigatório — com `*` eu deixaria o `retry/backoff.ts` de fora sem perceber, e é justamente esse tipo de arquivo escondido em subpasta que costuma esquecer de ser atualizado. Resumindo: `*` = um nível, `**` = qualquer profundidade; o erro mais comum é usar `*` achando que está olhando tudo.

---

## Exemplo 2 — Achar todos os testes, em qualquer profundidade

**Pattern (Glob):** `tests/**/*.test.ts`

**Resultado esperado:**
```
tests/billing.test.ts
tests/api/webhook.test.ts
tests/services/mailer.test.ts
```

**Por que esse pattern:** os testes estão espalhados em níveis diferentes (um na raiz de `tests/`, outros em subpastas). Como `**` casa zero ou mais diretórios, um único pattern pega `tests/billing.test.ts` (zero níveis) e `tests/api/webhook.test.ts` (um nível). Usar `tests/*.test.ts` traria só o primeiro; usar `**/*.test.ts` sem o prefixo `tests/` funcionaria aqui, mas varreria `node_modules/` se ele existir — mais lento e cheio de lixo. Ancorar em `tests/` mantém o escopo exato.

---

## Exemplo 3 — Chaves `{}` para pegar vários formatos de configuração

**Pattern (Glob):** `config/*.{yaml,yml,json}`

**Resultado esperado:**
```
config/default.yaml
config/production.yaml
config/staging.yaml
```

**Por que esse pattern:** o time mistura `.yaml` e `.yml` ao longo do tempo, e às vezes aparece um `.json`. Com as chaves, uma busca só cobre as três extensões, em vez de três buscas separadas. Usei `*` (não `**`) de propósito: configuração mora só em `config/`, e não quero que um `fixtures/exemplo.yaml` dentro de `tests/` entre na lista e me faça editar o arquivo errado.

---

## Exemplo 4 — Grep simples com filtro por tipo: quem lê variável de ambiente?

**Pattern (Grep):** `process\.env\.[A-Z_]+` com `type: ts`, `output_mode: content`, `-n: true`

**Resultado esperado:**
```
src/env.ts:4:  const dbUrl = process.env.DATABASE_URL;
src/env.ts:5:  const smtpHost = process.env.SMTP_HOST;
src/services/mailer.ts:12:  apiKey: process.env.MAIL_API_KEY,
src/api/handlers/webhook.ts:9:  const secret = process.env.WEBHOOK_SECRET;
```

**Por que esse pattern:** o `\.` escapa o ponto (sem a barra invertida, o `.` casaria *qualquer* caractere e traria falsos positivos como `process_env`). `[A-Z_]+` limita ao padrão de nome de variável em maiúsculas, o que evita pegar `process.env` sozinho em comentários ou em um `const env = process.env` genérico. O filtro `type: ts` ignora `package.json`, `.md` e `.sql`, e o `-n` me dá o número da linha para abrir direto no ponto certo. O objetivo real aqui é inventariar quais variáveis o serviço exige antes de subir um ambiente novo — por isso quero a lista completa com linha, não só os nomes de arquivo.

---

## Exemplo 5 — `-i` (ignorar maiúsculas/minúsculas): descobrir onde o status "cancelado" é tratado

**Pattern (Grep):** `cancelad[oa]` com `-i: true`, `glob: src/**/*.ts`, `output_mode: content`, `-n: true`

**Resultado esperado:**
```
src/services/billing.ts:41:  if (fatura.status === 'CANCELADO') return;
src/services/billing.ts:88:  // fatura Cancelada não gera cobrança
src/api/handlers/boleto.ts:27:  status: 'cancelado',
src/services/mailer.ts:63:  const assunto = `Cobrança cancelada - ${numero}`;
```

**Por que `-i` faz diferença aqui:** esse é um caso real de código que cresceu com várias mãos. O mesmo conceito aparece como `'CANCELADO'` (constante vinda de uma coluna do banco, em caixa alta), `'cancelado'` (string de resposta da API), `Cancelada` (comentário escrito por humano, no feminino) e `cancelada` (texto de e-mail). Sem `-i`, uma busca por `cancelado` acharia só 1 das 4 ocorrências, e eu poderia concluir erradamente que o status é tratado em um único lugar — e deixar passar um bug onde um dos ramos compara com a caixa errada. O `[oa]` cobre o gênero (cancelado/cancelada) no mesmo pattern. O `glob` restringe a `src/`, para não vir ruído de `tests/` e de `migrations/` (que também falam em "cancelado").

---

## Exemplo 6 — `-C` (contexto nos dois lados): entender como um `TODO` de retry está escrito

**Pattern (Grep):** `TODO.*retry` com `-C: 3`, `output_mode: content`, `-n: true`

**Resultado esperado:**
```
src/services/billing.ts-52-  try {
src/services/billing.ts-53-    await gateway.charge(fatura);
src/services/billing.ts-54-  } catch (err) {
src/services/billing.ts:55:    // TODO: implementar retry com backoff exponencial
src/services/billing.ts-56-    logger.error('falha na cobrança', err);
src/services/billing.ts-57-    throw err;
src/services/billing.ts-58-  }
```

**Por que `-C 3` e não outro número:** um `TODO` sozinho não diz nada; o que importa é *o que o código faz ao redor*. Aqui, 3 linhas antes mostram a abertura do `try` e a chamada que pode falhar (`gateway.charge`), e 3 linhas depois mostram o que acontece hoje na falha (loga e relança). Com `-C 1` eu veria só o `catch` e o `logger`, sem saber qual chamada está protegida. Com `-C 10` o resultado engoliria funções vizinhas e eu teria que ler muito mais para achar o essencial. Para um bloco `try/catch` típico, 3 linhas de cada lado costuma ser a janela que pega o corpo inteiro. Usei `-C` (simétrico) porque a informação relevante está *dos dois lados* da linha encontrada.

---

## Exemplo 7 — `-A` e `-B` (contexto assimétrico): achar o nome de uma função a partir do erro

### 7A — `-B` para ver o que vem *antes*

**Pattern (Grep):** `throw new Error\('assinatura inválida'\)` com `-B: 6`, `path: src/api/handlers/`

**Resultado esperado:**
```
src/api/handlers/webhook.ts-14-export function validarWebhook(req: Request) {
src/api/handlers/webhook.ts-15-  const assinatura = req.headers['x-nexi-signature'];
src/api/handlers/webhook.ts-16-  const esperado = hmac(req.rawBody, SECRET);
src/api/handlers/webhook.ts-17-
src/api/handlers/webhook.ts-18-  if (assinatura !== esperado) {
src/api/handlers/webhook.ts:19:    throw new Error('assinatura inválida');
```

**Por que `-B 6`:** estou olhando um `throw` e quero saber *qual condição o dispara* e *em qual função ele vive*. Isso está sempre acima da linha. `-B 6` chega até a assinatura da função (`validarWebhook`) sem puxar nada que veio depois, que seria irrelevante para essa pergunta.

### 7B — `-A` para ver o que vem *depois*

**Pattern (Grep):** `export (async )?function enviarEmail` com `-A: 8`, `path: src/services/mailer.ts`

**Resultado esperado:**
```
src/services/mailer.ts:30:export async function enviarEmail(para: string, assunto: string) {
src/services/mailer.ts-31-  const payload = {
src/services/mailer.ts-32-    to: para,
src/services/mailer.ts-33-    subject: assunto,
src/services/mailer.ts-34-    from: FROM_PADRAO,
src/services/mailer.ts-35-  };
src/services/mailer.ts-36-  const resp = await client.send(payload);
src/services/mailer.ts-37-  return resp.id;
src/services/mailer.ts-38-}
```

**Por que `-A 8`:** aqui o que me interessa é o *corpo* da função, que vem depois da linha da declaração. Uma função curta como essa cabe em ~8 linhas, então `-A 8` mostra a função inteira (até o `}`). Se a função fosse longa, eu aumentaria; se eu só quisesse a assinatura e o primeiro bloco, `-A 3` bastaria. `-B` e `-A` existem separados justamente para eu não pagar por contexto que não vou usar.

---

## Resumo: qual ferramenta e qual pattern, quando

| Preciso de... | Ferramenta | Pattern típico |
|---|---|---|
| Arquivos só de um nível | Glob | `src/services/*.ts` |
| Arquivos em qualquer profundidade | Glob | `src/services/**/*.ts` |
| Várias extensões de uma vez | Glob | `config/*.{yaml,yml,json}` |
| Linhas com um texto exato / regex | Grep | `process\.env\.[A-Z_]+` |
| Variações de caixa/gênero no mesmo termo | Grep `-i` | `cancelad[oa]` |
| Entender o entorno de um achado (simétrico) | Grep `-C N` | `TODO.*retry` com `-C 3` |
| Ver o que vem antes (condição, assinatura) | Grep `-B N` | `throw new Error(...)` com `-B 6` |
| Ver o que vem depois (corpo da função) | Grep `-A N` | `function enviarEmail` com `-A 8` |

**Fluxo comum no dia a dia:** Glob para descobrir *onde* olhar (ou restringir com `glob`/`type` no Grep) → Grep para achar *o quê* → leitura do arquivo só nas linhas que importam.
