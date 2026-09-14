# `developer.url` ausente trava o deploy pra sempre

> **Plataforma:** Fluig · Central de Componentes · widget `.war`
> **Status:** confirmado · a falha acontece numa fila, não na tela

## Sintoma

Subo o `.war` na Central de Componentes. A tela diz **"enviado com sucesso, deploy
executando"** — e **nunca notifica**. Fica assim para sempre.

Parece servidor travado. Não é.

## Causa

O campo `developerUrl` do `WCMWidget` é **`@NotNull`**. Se o `application.info` não tem
`developer.url`:

1. O WildFly deploya o `.war` **normalmente** — por isso não parece erro de deploy;
2. `WCMDeployMDBBean.initApplication` → `WCMWidgetDAO.create` levanta
   `ConstraintViolationException` → rollback do persist;
3. O **ActiveMQ reenvia em loop** (`RESEND 1, 2, 3...`).

Ou seja: falha silenciosa numa fila. A tela nunca fica sabendo.

## Correção

Incluir no `application.info`:

```
developer.code=...
developer.name=...
developer.url=...
```

Os três. `developer.url` é o que costuma faltar.

## Como detectar antes

Baixar o log do servidor (Painel de Controle) e procurar:

```
WCMDeployMDBBean
ConstraintViolationImpl
propertyPath=
```

> **O `propertyPath` diz exatamente qual campo está faltando.** Serve para qualquer outra
> constraint, não só esta — é o atalho que transforma "deploy travado" em "falta o campo X".

**Prevenção que fecha o assunto:** validar no script de empacotamento, antes de gerar o `.war`
— `developer.url` presente, `application.code` == context-root == nome do war, `view.file`
existe, resource declarado existe. Erro de empacotamento vira erro de build, não mistério de
servidor.
