# Body do POST no FWRest não é parâmetro

> **Plataforma:** Protheus · AdvPL/TLPP · `FWRest`
> **Status:** confirmado · fonte: TDN + regras TOTVS

## Sintoma

`FWRest:Post()` sai com **body vazio**. A API do outro lado responde 400 dizendo que faltou
campo obrigatório — e o JSON estava montado certinho na variável.

Variantes do mesmo dia ruim:

- A API responde **202 Accepted** ou **204 No Content** e o AdvPL trata como **falha**.
- A API devolveu 400 com a explicação no corpo, e `GetResult()` volta **vazio** — o motivo
  do erro se perde.
- Precisa de **PATCH** e não acha o método.

## Causa

Quatro comportamentos não óbvios da mesma classe:

1. **`Post(aHeader)` só recebe o header.** O corpo tem que ser posto **antes**, com
   `SetPostParams(cBody)`. Já `Put()` e `Delete()` recebem o body como **2º argumento
   posicional** e ignoram `SetPostParams()`. As duas convenções convivem na mesma classe.
2. **Faixa de sucesso legada.** Por padrão (`SetLegacySuccess(.T.)`) só **200 e 201** contam
   como sucesso. 202, 204 e 207 caem no `Else`.
3. **`SetChkStatus(.T.)` (default) esconde o corpo do erro.** Para ler o body de um 4xx/5xx é
   preciso `SetChkStatus(.F.)` e inspecionar `GetHTTPCode()` + `GetResult()` na mão.
4. **`FWRest` não tem PATCH.** Só GET, POST, PUT e DELETE.

## Correção

```tlpp
oClient := FWRest():New("https://api.exemplo.com")  // só o host, sem path
oClient:SetPath("/api/v1/recurso")
oClient:SetPostParams(cBody)          // 1. o body vai aqui, ANTES do Post
oClient:SetLegacySuccess(.F.)         // 2. aceita 200-299   (LIB 20240812+)
oClient:SetChkStatus(.F.)             // 3. deixa ler 4xx/5xx (Release 23+)

If oClient:Post(aHeader)
    cCode := oClient:GetHTTPCode()    // caractere: "200", "404"
    cResp := oClient:GetResult()
EndIf
```

Para PATCH, `HTTPQuote()`.

E o header sempre com **espaço depois dos dois-pontos** — `"Content-Type: application/json"`.
O parser exige.

## Como detectar antes

> Se o `Post` "funciona" mas o outro lado diz que não recebeu nada, **o body nem saiu.**

Ligar `FWTraceLog=1` na seção do ambiente no `appserver.ini` e ler a requisição crua —
mata a dúvida em um teste.

Ao integrar com API que não é TOTVS, checar antes quais status ela usa. Se a documentação
menciona 202 ou 204, já entrar com `SetLegacySuccess(.F.)`.

⚠️ `GetHTTPCode()` pode vir **vazio** em 204 sem *reason phrase*.
