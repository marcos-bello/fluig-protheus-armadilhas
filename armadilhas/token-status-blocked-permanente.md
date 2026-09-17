# Status BLOCKED nunca volta para ACTIVE

> **Plataforma:** portal público com login por token · Java
> **Status:** confirmado em auditoria · defeito de disponibilidade, não de segurança

## Sintoma

O usuário erra o token 5 vezes, espera os 15 minutos que o sistema "promete", tenta de novo e
recebe **"CNPJ ou token inválido"** — a mesma mensagem de token errado.

Não há como ele saber que está bloqueado. Abre chamado dizendo que o token não funciona.

## Causa

Duas coisas somadas, e as duas parecem certas isoladamente.

**1. A ordem dos `if` mata o desbloqueio.** O repositório grava, na 5ª falha,
`status = 'BLOCKED'` e `blocked_until = agora + 15min`. Mas a validação testa nesta ordem:

```java
if (!record.isActive())      return erro(401, "INVALID_CREDENTIALS", "CNPJ ou token inválido.");
// ...
if (record.isBlocked(now) || record.getAttemptCount() >= record.getMaxAttempts())
    return erro(429, "TOKEN_LOCKED", "Muitas tentativas. Solicite um novo token.");
```

`isActive()` só é `true` para status `ACTIVE`. Como o status já virou `BLOCKED`, a execução
**para na primeira linha** e nunca chega na segunda.

Consequências:

- o `blocked_until` **nunca é lido** — os 15 minutos não existem;
- o ramo `429 TOKEN_LOCKED` é **código inalcançável**;
- e nada, em lugar nenhum, devolve `BLOCKED` → `ACTIVE`.

**2. O bloqueio é permanente.** A única saída é reemitir o token pelo processo.

## Por que passa despercebido

Porque o schema tem `blocked_until` e o código tem a mensagem de bloqueio: **tudo indica que o
desbloqueio temporário existe.** Ninguém testa a 6ª tentativa depois de esperar.

E não é falha de segurança — o bloqueio *funciona*, até demais. É **disponibilidade**, que é o
tipo de defeito que vira chamado em vez de incidente.

## Correção

Escolher uma das duas, conscientemente:

- **bloqueio temporário** → testar `isBlocked()` **antes** de `isActive()`, e ter uma rotina
  que devolva `BLOCKED` → `ACTIVE` quando `blocked_until` passar;
- **bloqueio permanente** → tirar `blocked_until` do schema, remover o ramo 429 e **dizer ao
  usuário** que ele precisa solicitar um token novo.

O pior cenário é o atual: prometer o temporário e entregar o permanente.

## Como detectar antes

- Ter o **procedimento de reemissão pronto** antes de qualquer portal entrar no ar.
- No teste de aceite, incluir sempre o caso **"6ª tentativa depois de esperar o bloqueio"** —
  não só "5 tentativas erradas bloqueiam".
- Ao revisar uma cadeia de `if` de validação, perguntar de cada ramo: **"o que precisa ser
  verdade para chegar aqui?"** Ramo inalcançável é sintoma de que a ordem dos portões está
  errada.

## Relacionadas

- [CC do notifier recebe o mesmo corpo do e-mail](notifier-cc-vaza-corpo.md) — outro defeito
  da mesma família de login por token
