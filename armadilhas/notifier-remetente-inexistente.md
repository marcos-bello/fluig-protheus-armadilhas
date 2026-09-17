# Remetente inexistente derruba o e-mail inteiro — e o erro é assíncrono

> **Plataforma:** Fluig · e-mail server-side · `notifier`
> **Status:** confirmado em servidor · o erro existe, só não volta para o seu script

## Sintoma

`notifier.notify(...)` retorna sem exceção. O log do script diz que aceitou, com template
e destinatários corretos. **Nenhum e-mail chega — nem para fora, nem para a cópia interna.**

A tentação é concluir "o notifier não entrega para quem não é usuário do tenant" ou "o
template não existe". Duas histórias que encaixam e estão erradas.

## Causa

Quatro milissegundos depois do `notify`, numa **thread assíncrona** (`EJB ASYNC`), o
servidor registra:

```
ERROR [com.fluig.ecm.notification.service.CustomNotificationServiceBean]
  Erro ao enviar o email: Usuário corrente não existe: (<login>)
  com.fluig.bpm.exception.validation.BPMSenderNotFoundException
    at CustomNotificationServiceBean.validate(...:201)
```

O **remetente** não existe no tenant. A validação roda antes de qualquer SMTP e aborta o
envio **para todos os destinatários de uma vez**. Como a exceção nasce em outra thread, ela
nunca volta para o script — o `notify` parece ter dado certo.

⚠️ `wcmadmin` é o superusuário do Fluig Cloud **em alguns tenants, não em todos**. Assumir
que existe é a origem mais comum desse erro.

## Correção

Usar um login que exista **naquele** tenant. Um padrão observado em ambientes cloud é o
e-mail com `@` e `.` convertidos em `.`, mais um sufixo numérico:

| E-mail | Login |
| --- | --- |
| `fulano@empresa.com.br` | `fulano.empresa.com.br.1` |

Mas isso varia por tenant — não deduza, confirme.

O remetente também precisa ser **diferente** do destinatário (`Same sender and receiver`).

## Como detectar antes

Nunca aceitar "notify aceito" como entrega. Depois de qualquer `notify`, procurar no log da
**mesma janela de segundos** por `CustomNotificationServiceBean`. O erro está lá, em outra
thread, e não aparece em lugar nenhum da interface.

Para descobrir o login válido sem acesso ao admin, o log do servidor entrega o padrão real:

```bash
grep -oE 'login=[a-z0-9._@-]+' server.log | sort -u
```

## Relacionadas

- [`notifier.notify` engole e-mail em silêncio](notifier-email-silencioso.md)
- [CC do notifier recebe o mesmo corpo do e-mail](notifier-cc-vaza-corpo.md)
