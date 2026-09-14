# Armadilhas Fluig & Protheus

Falhas reais de plataforma TOTVS — **Fluig** (BPM/ECM/WCM) e **Protheus** (AdvPL/TLPP) —
documentadas por quem as levou na cara, com a evidência que confirmou a causa.

Não é uma coletânea de dicas. Cada nota aqui custou de algumas horas a algumas semanas, e
existe porque a documentação oficial não cobre o caso, cobre numa linha que ninguém acha
antes, ou — em pelo menos dois casos — porque o comentário no código dizia o oposto do que
o servidor fazia.

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

## Fluig

| Armadilha | O que quebra |
|---|---|
| [`ArrayList` do Java não tem `.length` no Rhino](armadilhas/rhino-arraylist-length.md) | E-mail nunca sai; guarda de lista sempre verdadeira |
| [Tabela pai-filho sem `tablename` não persiste](armadilhas/form-pai-filho-tablename.md) | Engine aceita o `cardData`, grava o card, e a tabela filha volta vazia |
| [Campo `disabled` não é gravado — e o `fieldset` leva o painel inteiro](armadilhas/form-disabled-nao-grava.md) | Dados somem ao devolver a tarefa |
| [Nova versão de formulário não alcança solicitação em andamento](armadilhas/form-versao-nao-retroage.md) | Você publica a correção e a solicitação presa continua presa |
| [Scripts de evento do mesmo processo compartilham escopo Rhino](armadilhas/eventos-escopo-rhino.md) | Helpers de arquivos diferentes se sobrescrevem em silêncio |
| [Template literal engole a tradução i18n](armadilhas/i18n-template-literal.md) | Tradução vira vazio dentro de backtick, sem erro |
| [Emoji de 4 bytes no evento derruba a publicação](armadilhas/publish-emoji-4-bytes.md) | `ARJUNA016053` sem stack, sem campo, sem pista |
| [`developer.url` ausente trava o deploy pra sempre](armadilhas/deploy-developer-url.md) | "Deploy executando" que nunca termina |

## Protheus

| Armadilha | O que quebra |
|---|---|
| [Compilador só aceita CP1252](armadilhas/protheus-encoding-cp1252.md) | Fonte gerado por editor moderno não compila, ou vira mojibake |
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
