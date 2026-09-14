# Campo `disabled` não é gravado — e o `fieldset` leva o painel inteiro

> **Plataforma:** Fluig · formulário · `displayFields` / `enableFields`
> **Status:** confirmado · causa isolada em 30 segundos pelo recorte da tag

## Sintoma

A solicitação é aberta com **todos** os campos preenchidos. Alguém pega a tarefa do
papel/grupo, vê que não é da competência dela e **devolve** (ou cancela). Quando a
solicitação reaparece, **os dados sumiram** — e ninguém editou nada.

O que engana: os dados de cabeçalho (nome, empresa, matrícula) continuam lá. Só somem o
motivo, a planilha inteira e as observações. Parece perda seletiva, parece bug do Fluig,
parece problema de permissão.

Não é. É exatamente o recorte de um `<fieldset>`.

## Causa

O `displayFields` bloqueava o painel nas atividades finais assim:

```js
customHTML.append('document.getElementById("panel_solicitacao").disabled = true;');
```

`panel_solicitacao` é um `<fieldset>`. Pela especificação HTML, **`fieldset` desabilitado
desabilita todos os controles dentro dele** — e **controle desabilitado não é enviado no
POST**. O Fluig recebe o formulário sem aqueles campos e grava o registro em branco.

A TDN diz com todas as letras (*Eventos de Formulário*, pageId 270924158, seção
`enableFields`):

> "não é permitido utilizar a propriedade **disabled**, pois os campos não serão gravados ao
> salvar o registro de formulário. Para esta situação, deve-se utilizar a propriedade
> **readonly**."

E o fórum Fluiggers, no tópico *"Formulário perdendo os valores após certa etapa do fluxo"*
(2298): *"se deixar como disabled o Fluig salvará em branco"*.

## Correção

**Nunca `disabled`. `readonly`.** Campo `readonly` vai no POST em todos os cenários.

Bloqueio completo de um painel, sem tirar nada do POST:

| elemento | como travar |
| --- | --- |
| `input`, `textarea` | `this.readOnly = true` + `tabindex="-1"` |
| `select` | `readonly` **não funciona** em select. Desabilitar as `<option>` **não** selecionadas e deixar o `<select>` habilitado — o valor continua indo no POST |
| `input type="date"` | `readonly` + esconder `::-webkit-calendar-picker-indicator` |
| zoom / autocomplete | `readonly` no input + esconder o botão de lupa + `stopImmediatePropagation` no clique |
| botões add/remove do pai×filho | esconder |

Aparência de campo bloqueado se resolve com CSS (`background-color: #eee`), não com `disabled`.

### Por que `enableFields` + `protect` não resolve tudo

`form.setEnabled(campo, false, true)` existe e protege o valor — **mas** o FormController
(pageId 662892312) avisa:

> "A função de proteção dos dados do formulário só é válida no contexto de uma **movimentação
> workflow**."

Ou seja, não cobre justamente **devolver a tarefa**, liberar, salvar rascunho e movimentação
por API — que é exatamente onde o dado estava sumindo.

E `enableFields` **prefixa `name` e `id` com `_`** (`motivo` vira `_motivo`), o que quebra todo
seletor jQuery do formulário naquelas atividades. Use só quando precisar de bloqueio à prova de
DevTools, sabendo do preço.

## Como detectar antes

- **F12 → Network → movimentar → olhar o payload do POST.** Campo que não aparece ali vai ser
  gravado vazio. Este é o teste definitivo, não a inspeção visual.
- No console: `document.getElementById('painel').disabled` e `$('#campo').is(':disabled')`.
- Ao revisar form alheio: `grep -n "disabled" *.js`. Qualquer `.disabled = true` em
  `<fieldset>`, `<div>` ou campo é suspeito imediato.
- **Desconfiar de "o que sobreviveu × o que sumiu".** Se o recorte da perda bate com um bloco
  do HTML, é bloqueio de container — não lógica de negócio. Foi assim que este caso caiu em
  30 segundos: listar os dois conjuntos e procurar a tag que os separa.

## Relacionadas

- [Tabela pai-filho sem `tablename`](form-pai-filho-tablename.md) — o POST vai completo e mesmo
  assim não persiste
- [Nova versão de formulário não alcança solicitação em andamento](form-versao-nao-retroage.md)
