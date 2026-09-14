# Template literal engole a tradução i18n

> **Plataforma:** Fluig · widget / Custom Element · FreeMarker + ES6
> **Status:** confirmado · fonte: convenções oficiais de customização Fluig

## Sintoma

A tradução não aparece. No lugar do texto vem string vazia, `undefined`, ou o JS quebra com
`i18n is not defined` — **só dentro de template literal (backtick)**. A mesma chave, na linha
de cima com aspas normais, funciona.

## Causa

Duas linguagens disputam a mesma sintaxe `${...}`:

- **FreeMarker** resolve `${...}` **no servidor**, antes de enviar o arquivo.
- **JavaScript ES6** resolve `${...}` **no navegador**, dentro de backticks.

Dentro de backticks quem ganha é o **JavaScript** — ele procura uma variável `i18n` em runtime,
que não existe no cliente. O FreeMarker nem chega a ver a expressão.

```javascript
// ❌ o JS avalia ${...} e procura i18n no browser
const msg = `${i18n.getTranslation('save.success')}: ${name}`;
```

## Correção

Usar a sintaxe alternativa do FreeMarker `[=...]`, que **não colide** com template literal:

```javascript
// ✅ [=...] resolve server-side, funciona até dentro de backticks
const msg = `[=i18n.getTranslation('save.success')]: ${name}`;
```

Alternativa, mantendo `${...}`: declarar a tradução numa **variável separada**, fora dos
backticks e entre aspas, e só depois concatenar.

```javascript
const successMsg = "${i18n.getTranslation('save.success')}";
const msg = `${successMsg}: ${name}`;
```

## Como detectar antes

> **Regra:** em arquivo `.js`, `${i18n...}` e backtick **não convivem**.
> Viu backtick, use `[=i18n...]`.

Busca barata no fonte — procurar backtick e `${i18n` na mesma linha:

```bash
grep -n '`.*\${i18n' *.js
```

Qualquer ocorrência é bug.

⚠️ **Não sair trocando `${...}` por `[=...]` no código legado.** Respeite o padrão já existente
no arquivo; `[=...]` é para código novo ou arquivo sem padrão definido. Misturar as duas
sintaxes no mesmo arquivo confunde mais do que resolve.

## Relacionadas

- [`ArrayList` do Java não tem `.length` no Rhino](rhino-arraylist-length.md) — o choque
  inverso: ES6 onde só cabe ES5
