# Nova versão de formulário não alcança solicitação em andamento

> **Plataforma:** Fluig · formulário · versionamento
> **Status:** confirmado · fonte: TDN + FAQ BPM-055

## Sintoma

O formulário foi corrigido e publicado. A solicitação que estava travada **continua travada,
com a mensagem antiga** — inclusive citando uma regra que já não existe em lugar nenhum do
código.

Num caso real a tela reclamava de uma validação de prazo que só existia numa cópia de três
meses antes. A correção estava em disco havia mais de um mês, e o servidor nunca a viu.

O engano natural: *"publiquei de novo e não pegou, então o deploy falhou"*. Não falhou.

## Causa

A TDN (*Documentos → Formulário*, pageId 233763113):

> "Alterações em formulários vinculados com processos só são efetivadas em **novas
> solicitações**; para as solicitações em andamento, se mantêm a mesma versão de formulário."

E a FAQ **BPM-055** (pageId 243658034) fecha a porta dos fundos:

> "Atualmente, o fluig **não possui uma ferramenta de conversão de versão de formulário**.
> Quando a solicitação foi convertida, o processo foi atualizado para a versão corrente, porém
> o formulário continua apontando para a versão anterior."

O card guarda o número da versão do formulário com que **nasceu** —
`getDocumentPropertyVersion()` devolve exatamente isso. Publicar nova versão cria a 2.000, a
3.000, e a solicitação velha continua lendo a 1.000.

Converter a solicitação para a nova versão do **processo** também não resolve: o processo muda,
o formulário não. E vale para o processo do mesmo jeito — *"as solicitações que estão em
andamento continuam com as configurações da versão antiga"*.

## Correção

Depende de **para quem** é a correção. São dois trabalhos diferentes.

**Se é para as próximas solicitações** — publique normal, nova versão. Acabou.

**Se é para desatolar uma solicitação que já existe**, há dois caminhos:

1. **Publicar sem versionar.** No controle de versão do formulário existe **"Manter versão"**:
   *"será mantida a versão atual do documento… trocar todo o conteúdo corrente por um novo
   atualizado"*. Como o conteúdo da versão que o card aponta é substituído, a solicitação em
   andamento passa a renderizar o código novo.
   ⚠️ A opção **some** se o formulário foi criado com *"Criar versão/revisão obrigatória"*
   assinalado — nesse caso, só o caminho 2.

2. **Movimentar por fora do browser.** `beforeSendValidate` é **JavaScript do formulário**: só
   roda na tela. A FAQ **BPM-047** (pageId 243011198) confirma que a movimentação em bloco
   executa o `validateForm` (servidor) e **não** o que é client-side. Logo, movimentação em lote
   ou a API de *process-management* passam por cima da crítica — desde que o processo não tenha
   um `validateForm` equivalente no servidor. Confira a pasta `events/` antes.

## Como detectar antes

- **Comparar o texto do erro com o código em disco.** Mensagem que não aparece em
  `grep -rn "trecho da mensagem"` na pasta viva = o servidor está numa versão anterior. Este
  teste custa 30 segundos e mata o caso.
- Antes de prometer prazo, perguntar: **"é para as novas ou para a que está presa?"** Só um dos
  dois é "publicar".

## Relacionadas

- [Tabela pai-filho sem `tablename`](form-pai-filho-tablename.md) — por isso todo teste de
  gravação tem que ser em solicitação nova
- [Campo `disabled` não é gravado](form-disabled-nao-grava.md)
