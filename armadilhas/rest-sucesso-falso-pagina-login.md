# Sucesso falso quando o Fluig responde a página de login

> **Plataforma:** Fluig · API REST · ambiente com SSO
> **Status:** confirmado · `r.ok` é `true` e mesmo assim nada aconteceu

## Sintoma

O portal público mostra **"Cadastro enviado"**, com painel verde e tudo.

**Nenhum processo nasce no Fluig.**

Nenhum erro no console vindo do seu código. O único sinal sutil: o **protocolo não aparece**
na tela de confirmação.

## Causa

Com SSO no caminho (TOTVS Identity ou outro IdP), a API REST do Fluig **não devolve 401 nem
403** para quem não tem sessão. Ela devolve:

```
HTTP 200
Content-Type: text/html; charset=UTF-8

<body onload="document.forms[0].submit()">
  <form action="https://<idp>/cloudpass/SPInitPost/receiveSSORequest/...">
    <input type="hidden" name="RelayState"  value="<a URL que você chamou>"/>
    <input type="hidden" name="SAMLRequest" value="PD94bWwg..."/>
```

> **`r.ok` é `true`.** É uma resposta 200 legítima — só que o corpo é a página de login, não
> a sua API.

E o código típico engole isso sem reclamar:

```javascript
try { data = t ? JSON.parse(t) : {}; } catch (e) { data = { raw: t }; }  // ← engole o HTML
if (!r.ok) { throw ... }                                                  // ← 200, não entra
return data;                                                              // ← "sucesso"
```

O `JSON.parse` falha, o `catch` guarda o HTML num campo `raw`, e a promise **resolve**.

## Correção

Duas checagens, ambas necessárias:

```javascript
var tipo = ('' + (r.headers.get('content-type') || '')).toLowerCase();

var pareceLogin = (tipo.indexOf('html') > -1) ||
                  (/SAMLRequest|SPInitPost|document\.forms\[0\]\.submit/i).test('' + t);
if (pareceLogin) { throw new Error('SEM_SESSAO: o servidor respondeu a página de login.'); }

// e nunca declarar sucesso sem o identificador de verdade
if (!extrairInstanceId(data)) { throw new Error('RESPOSTA_INESPERADA'); }
```

## A regra geral

> **`r.ok` não é prova de sucesso quando existe SSO no caminho.** Sucesso é **ter recebido o
> dado que você pediu** — aqui, o `processInstanceId`. Valide o `content-type` e exija o
> identificador.

Vale para qualquer chamada REST a partir de página pública nesse tipo de ambiente, não só
para o `/start`.

## Como detectar antes

- **Testar em aba anônima de verdade.** Logado, o cookie de sessão mascara tudo e o teste
  passa sem provar nada.
- Conferir no Fluig se o processo **realmente nasceu**. Não confiar na tela do portal.
- Diagnóstico que resolve em 10 segundos, no console:

```javascript
fetch(location.origin + '/process-management/api/v2/processes/<PROC>/start', {
  method: 'POST', credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ targetState:0, targetAssignee:'', subProcessTargetState:0,
                         comment:'teste', formFields:{} })
}).then(async function (r) {
  console.log('STATUS:', r.status, '| CONTENT-TYPE:', r.headers.get('content-type'));
  console.log('BODY:', (await r.text()).slice(0, 1500));
});
```

Se o `CONTENT-TYPE` vier `text/html`, você achou o problema.

## Relacionadas

- [`notifier.notify` engole e-mail em silêncio](notifier-email-silencioso.md) — outro
  "sucesso" que não é sucesso
