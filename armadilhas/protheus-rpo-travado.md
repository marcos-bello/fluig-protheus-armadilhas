# RPO travado impede compilar

> **Plataforma:** Protheus · compilação · AppServer
> **Status:** confirmado · a mensagem diz a verdade, se você ler literalmente

## Sintoma

A compilação falha com:

```
It wasn't possible to obtain exclusive access to the objects repository
```

O fonte está correto, a conexão está de pé, o servidor responde. Só não compila.

## Causa

O **RPO é o repositório de objetos do AppServer**, e a gravação exige **acesso exclusivo**.
Qualquer usuário logado no SmartClient — ou **qualquer JOB rodando** naquele ambiente —
segura o RPO.

Não é bug, e não adianta repetir: vai falhar igual enquanto alguém estiver dentro.

## Correção

Duas saídas:

1. **Derrubar quem está segurando** — desconectar usuários e JOBs do ambiente antes de
   compilar.
2. **Automatizar** — chave `BuildKillUsers` na seção `[GENERAL]` do `appserver.ini`, que
   desconecta automaticamente antes da compilação.

## Como detectar antes

> A mensagem fala de *"objects repository"*, não de permissão nem de rede. **É concorrência,
> não configuração.** Ler literalmente economiza a caçada errada.

Antes de uma janela de compilação em ambiente compartilhado, conferir quem está conectado.
Em homologação com JOB agendado o JOB reaparece sozinho — aí `BuildKillUsers` resolve de vez.

⚠️ Não confundir com o outro sintoma de compilação: caractere corrompido ou sintaxe inválida
numa linha visivelmente correta é **encoding**, não RPO →
[Compilador Protheus só aceita CP1252](protheus-encoding-cp1252.md).

## Relacionadas

- [Compilador Protheus só aceita CP1252](protheus-encoding-cp1252.md)
- [Janela dentro de transação trava o Protheus](protheus-janela-em-transacao.md)
