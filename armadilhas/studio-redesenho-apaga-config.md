# Redesenhar no Studio apaga a configuração da atividade

> **Plataforma:** Fluig · BPM · Studio · `.process` + `.ecm30.xml`
> **Status:** confirmado · a atividade sobrevive, a configuração dela não

## Sintoma

Você mexe no diagrama para acrescentar um desvio e um tratamento de erro. Depois, uma tarefa
**que já estava configurada** aparece sem mecanismo de atribuição — e as tarefas novas nascem
**sem responsável nenhum**.

Publicado assim, a tarefa nasce travada: ninguém a recebe e ninguém consegue pegá-la.

## Causa

No `.process`, a atividade que sobreviveu ao redesenho fica reduzida a isto:

```xml
<bpmn2:BpmnTask id="task2" name="Analisar Cadastro"
    incoming="flow1" outgoing="flow4" type="80" extendedFields="&lt;list/&gt;"/>
```

Sumiram `managerMechanism`, `managerAssignmentControllerString`, `prazoConclusao`,
`expediente`, `authNotify` — tudo. As atividades novas nascem só com os defaults, também sem
atribuição.

> ⚠️ **Vale para service task também**: elas nascem **sem `scriptFileName`**. O motor chama a
> função pelo nome do id da atividade e, sem o arquivo apontado, a atividade "passa" sem
> executar nada — sem erro, sem log, sem nada.

## Correção

Reescrever a configuração nos **dois** arquivos. Eles não se sincronizam sozinhos e usam
**sintaxes diferentes para a mesma coisa**.

**`.process`** — atributos do elemento:

```xml
managerMechanism="Pool Grupo"
managerAssignmentControllerString="&lt;org.eclipse.bpmn2.impl.AssignmentControllerPoolGroup&gt;&#10;
  &lt;groupId&gt;MEU_GRUPO&lt;/groupId&gt;&#10;
  &lt;mechanismName&gt;Pool Grupo&lt;/mechanismName&gt;&#10;
&lt;/org.eclipse.bpmn2.impl.AssignmentControllerPoolGroup&gt;"
prazoConclusao="240.0"
scriptFileName="MeuProcesso.servicetask13.js"   (só service task)
```

**`.ecm30.xml`** — elementos do `ProcessState`:

```xml
<engineAllocationId>Pool Grupo</engineAllocationId>
<engineAllocationConfiguration>&lt;AssignmentController&gt;&lt;Group&gt;MEU_GRUPO&lt;/Group&gt;&lt;/AssignmentController&gt;</engineAllocationConfiguration>
<deadlineTime>14400</deadlineTime>
```

E, para service task, um `<WorkflowProcessEvent>` com `eventId` = id da atividade e o **corpo
do script inteiro** dentro de `<eventDescription>`.

### Duas pegadinhas de formato

> **As unidades são diferentes nos dois arquivos.** `deadlineTime` do `.ecm30.xml` é em
> **segundos**; `prazoConclusao` do `.process` é em **minutos**. 14400 s = 240 min = 4 h.
> A razão é constante: 60.

> **O `.process` é `encoding="ASCII"`.** Acento em rótulo de fluxo tem que virar entidade
> **hexadecimal** — `Pend&#xea;ncia`, não `Pendência`.

## O desvio também nasce quebrado

O Studio marca o losango com o decorador de erro (círculo vermelho). O motivo: o desvio nasce
com **uma** condição-placeholder que não aponta para lugar nenhum, mesmo tendo duas saídas.

| Arquivo | Como nasce | Como tem que ficar |
| --- | --- | --- |
| `.process` | 1 `ConditionImpl`, `<expression>TRUE</expression>`, `<targetTask></targetTask>` **vazio** | 1 `ConditionImpl` por saída, cada uma com `targetTask` = id do destino |
| `.ecm30.xml` | 1 `<ConditionProcessState>` com `<condition>TRUE</condition>` e `<destinationSequenceId>0</destinationSequenceId>` | 1 por saída, com `destinationSequenceId` = sequence do destino |

Escape do atributo `condition` no `.process`: `<` → `&lt;` · `>` → `&gt;` · quebra de linha →
`&#10;` · aspas dentro da expressão → `&amp;quot;` (duplo, porque o XML interno já escapa e
depois o atributo escapa de novo).

⚠️ A `<version>` da PK em `ConditionProcessState` é a do **ProcessDefinitionVersion**, não a do
`ProcessState`. Copiar da condição que já existe no arquivo.

## Como detectar antes

Depois de **qualquer** mexida no diagrama, rodar uma verificação antes de publicar:

```python
# toda tarefa de usuário (bpmnType 80) tem que ter alocação
for m in re.finditer(r'<ProcessState\b.*?</ProcessState>', ecm30, re.S):
    d = dict(re.findall(r'<(\w+)>([^<]*)</\1>', m.group(0)))
    if d.get('bpmnType') == '80' and not d.get('engineAllocationId', '').strip():
        print('SEM MECANISMO:', d.get('sequence'), d.get('stateName'))
```

E, para service task (`bpmnType 82`), conferir que existe `scriptFileName` no `.process`
**e** `WorkflowProcessEvent` no `.ecm30.xml`.

Vale automatizar a checagem inteira: XML bem-formado, `.process` em ASCII puro, tarefa de
usuário sem mecanismo, service task sem script, desvio com `targetTask` vazio ou com número
de condições diferente do número de saídas, e dessincronia entre os dois arquivos.

⚠️ Editar o `.js` **não** atualiza o `<eventDescription>` do `.ecm30.xml`. Sem sincronizar, o
que sobe é a versão velha do script.

## Relacionadas

- [Scripts de evento do mesmo processo compartilham escopo Rhino](eventos-escopo-rhino.md)
- [Nova versão de formulário não alcança solicitação em andamento](form-versao-nao-retroage.md)
