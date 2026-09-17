# Janela dentro de transação trava o Protheus

> **Plataforma:** Protheus · AdvPL/TLPP · transação, MVC, ponto de entrada
> **Status:** confirmado · regra SonarQube TOTVS CA1002 (MAJOR)

## Sintoma

A rotina **congela**. Nenhuma mensagem, nenhum erro no log — o usuário fica olhando a tela
parada e os registros seguem **bloqueados para todo mundo**.

Em JOB é pior: não há ninguém para clicar em nada, e o processo fica preso até derrubar o
AppServer.

Aparece muito em **ponto de entrada de gravação** (`MT100GRV`, `MT100APP`) e no commit de MVC.

## Causa

Chamada de interface dentro de `Begin Transaction` / `End Transaction`, ou dentro do commit
do modelo MVC:

`MsgAlert()` · `MsgYesNo()` · `MsgInfo()` · `Aviso()` · `Help()` · `Pergunte()` · `ParamBox()`

A transação segura os locks e fica esperando um clique. Enquanto o clique não vem — e em JOB
não vem nunca — a transação não fecha e os registros continuam travados.

## Correção

Tirar a tela de dentro da transação: **decidir antes, gravar depois.**

```
// errado
Begin Transaction
    If MsgYesNo("Confirma?")   // <<< trava aqui
        ...
    EndIf
End Transaction

// certo
lConfirma := MsgYesNo("Confirma?")   // pergunta ANTES
If lConfirma
    Begin Transaction
        ...
    End Transaction
EndIf
```

Para reportar o que aconteceu dentro da transação, acumular numa variável e mostrar **depois**
do `End Transaction`. Para log, `FWLogMsg()` — nunca `ConOut()`.

## Como detectar antes

> **Ponto de entrada com nome de gravação roda dentro da transação da rotina padrão** — mesmo
> que não exista nenhum `Begin Transaction` escrito no seu fonte. `MT100GRV` e `MT100APP` são
> os clássicos.

Revisão barata: procurar `Msg`, `Aviso`, `Help`, `Pergunte`, `ParamBox` no fonte e, para cada
ocorrência, perguntar **"isto está dentro de transação, ou dentro de um PE de gravação?"**

Mesmo raciocínio para laço: `GetMV()`, `SuperGetMV()` e `ExistBlock()` **dentro de
`While`/`For`** degradam performance — cachear antes do laço.

## Relacionadas

- [RPO travado impede compilar](protheus-rpo-travado.md)
