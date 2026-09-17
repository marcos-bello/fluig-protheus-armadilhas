# Armadilhas Fluig & Protheus

Falhas reais de plataforma TOTVS — **Fluig** (BPM/ECM/WCM) e **Protheus** (AdvPL/TLPP) —
documentadas por quem as levou na cara, com a evidência que confirmou a causa.

Não é uma coletânea de dicas. Cada nota aqui custou de algumas horas a algumas semanas, e
existe porque a documentação oficial não cobre o caso, cobre numa linha que ninguém acha
antes, ou — em pelo menos dois casos — porque o comentário no código dizia o oposto do que
o servidor fazia.

**20 armadilhas** publicadas.

---

## Por que o formato é sempre o mesmo

```
Sintoma  →  o que você vê, incluindo o que te leva à conclusão errada
Causa    →  o mecanismo, com a fonte ou o teste que provou
Correção →  o que fazer
Detectar →  o grep, o teste ou a pergunta que pega isso antes de custar caro
```

A seção que importa é a última. Corrigir resolve uma vez. **"Como detectar antes"
resolve as próximas** — e é o que transforma um incidente em regra de revisão.

Quando a causa não foi provada, a nota diz `hipótese, não testada` e qual teste falta.
Nenhuma afirmação aqui é palpite apresentado como fato.

---

## Fluig — Formulário

| Armadilha | O que quebra |
|---|---|
| [Tabela pai-filho sem `tablename` não persiste](armadilhas/form-pai-filho-tablename.md) | Engine aceita o `cardData`, grava o card, e a tabela filha volta vazia |
| [Campo `disabled` não é gravado — e o `fieldset` leva o painel inteiro](armadilhas/form-disabled-nao-grava.md) | Dados somem ao devolver a tarefa |
| [Nova versão de formulário não alcança solicitação em andamento](armadilhas/form-versao-nao-retroage.md) | Você publica a correção e a solicitação presa continua presa |

## Fluig — Processo, eventos e Studio

| Armadilha | O que quebra |
|---|---|
| [Redesenhar no Studio apaga a configuração da atividade](armadilhas/studio-redesenho-apaga-config.md) | Tarefa nasce sem responsável; service task "passa" sem executar nada |
| [Scripts de evento do mesmo processo compartilham escopo Rhino](armadilhas/eventos-escopo-rhino.md) | Helpers de arquivos diferentes se sobrescrevem em silêncio |
| [`ArrayList` do Java não tem `.length` no Rhino](armadilhas/rhino-arraylist-length.md) | Guarda de lista sempre verdadeira; o e-mail nunca sai |

## Fluig — E-mail e notificação

| Armadilha | O que quebra |
|---|---|
| [`notifier.notify` engole e-mail em silêncio](armadilhas/notifier-email-silencioso.md) | O log diz "enviado a 5 destinatários" e ninguém recebe |
| [Remetente inexistente derruba o e-mail inteiro](armadilhas/notifier-remetente-inexistente.md) | Erro real existe — em outra thread, invisível ao script |
| [CC do notifier recebe o mesmo corpo do e-mail](armadilhas/notifier-cc-vaza-corpo.md) | A "cópia de cortesia" entrega a credencial de terceiro |

## Fluig — Dataset

| Armadilha | O que quebra |
|---|---|
| [Dataset sincronizado que é job não pode devolver linhas](armadilhas/dataset-job-nao-devolve-linhas.md) | `Cannot insert duplicate row` num job que funcionou perfeitamente |

## Fluig — Página pública e integração

| Armadilha | O que quebra |
|---|---|
| [Sucesso falso quando o Fluig responde a página de login](armadilhas/rest-sucesso-falso-pagina-login.md) | `r.ok` é `true`, o portal diz "enviado", e nenhum processo nasce |
| [Status BLOCKED nunca volta para ACTIVE](armadilhas/token-status-blocked-permanente.md) | Bloqueio "temporário" de 15 min que na verdade é permanente |

## Fluig — i18n, publicação e deploy

| Armadilha | O que quebra |
|---|---|
| [Template literal engole a tradução i18n](armadilhas/i18n-template-literal.md) | Tradução vira vazio dentro de backtick, sem erro |
| [Emoji de 4 bytes no evento derruba a publicação](armadilhas/publish-emoji-4-bytes.md) | `ARJUNA016053` sem stack, sem campo, sem pista |
| [`developer.url` ausente trava o deploy pra sempre](armadilhas/deploy-developer-url.md) | "Deploy executando" que nunca termina |

## Protheus

| Armadilha | O que quebra |
|---|---|
| [Body do POST no FWRest não é parâmetro](armadilhas/protheus-fwrest-body-post.md) | POST sai vazio; 202/204 tratados como falha; corpo do erro some |
| [Janela dentro de transação trava o Protheus](armadilhas/protheus-janela-em-transacao.md) | Rotina congela e os registros ficam bloqueados para todo mundo |
| [Compilador só aceita CP1252](armadilhas/protheus-encoding-cp1252.md) | Fonte gerado por editor moderno não compila, ou vira mojibake |
| [RPO travado impede compilar](armadilhas/protheus-rpo-travado.md) | "Exclusive access" — é concorrência, não configuração |
| [Query sem `D_E_L_E_T_` traz registro apagado](armadilhas/protheus-query-deletado.md) | Relatório com número errado e nenhum erro |

---

## Ambiente

As notas foram levantadas em **Fluig 1.8.2 (Crystal Mist)** e **2.0 (Voyager)**, com
**MySQL** e **Protheus 12**. Comportamento de plataforma tende a valer em qualquer tenant,
mas onde a nota depende de versão, charset de coluna ou banco, ela diz.

## Escopo

Só comportamento de **plataforma**. Nada aqui contém código, dado, nome de cliente ou
identificador de projeto de terceiro — os incidentes estão descritos pelo mecanismo, não
pelo contexto em que apareceram.

## Contribuindo

Achou uma armadilha? Abra uma issue com sintoma, causa e — principalmente — **como você
provou a causa**. Correção sem prova eu não publico; é assim que nasce o próximo mito de
plataforma.

## Licença

[CC BY 4.0](LICENSE) — use, adapte e publique, citando a fonte.
