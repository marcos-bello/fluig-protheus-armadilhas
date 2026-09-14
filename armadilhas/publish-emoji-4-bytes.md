# Emoji de 4 bytes no evento derruba a publicação

> **Plataforma:** Fluig · publicação pelo Studio · MySQL `utf8mb3`
> **Status:** confirmado e resolvido · o erro não diz absolutamente nada

## Sintoma

Ao publicar o **formulário** (e depois o **processo**) o Studio devolve:

```
ARJUNA016053: Could not commit transaction.
```

Sem stack, sem nome de campo, sem indicação do arquivo. O mesmo projeto **já havia publicado
com sucesso horas antes**. Republicar não resolve, reiniciar o Studio não resolve, e o erro se
repete idêntico.

## Causa

> **Um caractere Unicode de 4 bytes (plano astral) escrito num comentário de `events/*.js` ou
> de `workflow/scripts/*.js`.** A coluna do MySQL é **`utf8mb3`** e não comporta 4 bytes — o
> INSERT falha e a transação inteira do publish cai.

O Studio manda os arquivos do projeto por **dois caminhos diferentes**, e só um deles esbarra
no charset da coluna:

| Arquivo | Vai como | Charset importa? |
|---|---|---|
| `HTML`, `custom.js` | `Attachment.filecontent : byte[]` | ❌ opaco ao charset |
| `events/*.js` | `CardEventDto.eventDescription : String` | ✅ **vira TEXTO em coluna** |
| `workflow/scripts/*.js` | idem | ✅ **vira TEXTO em coluna** |

Por isso **formulário e processo falham igual** — os scripts do workflow caem na mesma classe
de coluna.

O mesmo servidor já havia recusado esse byte antes, aí sim com mensagem clara:

```
Incorrect string value: '\xF0\x9F\x94\x97' for column 'DSL_EVENT' at row 1
   at com.mysql@5.1.49 ...
```

`\xF0\x9F\x94\x97` é 🔗.

### O que passa e o que não passa

| Classe | Bytes em UTF-8 | Resultado |
|---|---|---|
| Acentos (`ç`, `ã`, `é`) | 2 | ✅ passa |
| `⚠`, `→`, `═`, `—` | 3 | ✅ passa |
| Emoji astral: 🔑 🔴 🔗 📄 | **4** | ❌ **derruba o publish** |

> A confusão é fácil: `⚠️` (U+26A0) tem 3 bytes e passa; `🔑` (U+1F511) tem 4 e não passa. Os
> dois "parecem emoji" no editor.

## Correção

Trocar por ASCII nos arquivos que viram coluna — `🔑` → `>>>`, `🔴` → `[!]`. No caso real foram
cinco ocorrências, em quatro arquivos de evento e service task.

Varredura que fecha o assunto antes de publicar:

```python
import glob
for f in (glob.glob("forms/*/events/*.js") + glob.glob("forms/*/custom.js")
          + glob.glob("workflow/scripts/*.js") + glob.glob("datasets/*/*.js")):
    ruins = sorted({c for c in open(f, encoding="utf-8").read() if ord(c) > 0xFFFF})
    if ruins:
        print(f, [hex(ord(c)) for c in ruins])
```

`ord(c) > 0xFFFF` é exatamente o corte: acima disso o UTF-8 usa 4 bytes.

## Como detectar antes

- **Rodar a varredura acima antes de todo publish.** É instantânea, e o custo de não rodar é
  uma tarde.
- Em ~1.900 `events/*.js` publicados num workspace, o arquivo com emoji astral era **um só**. O
  padrão do resto do mundo já é não usar — só quebra quem inova no comentário.
- **Desconfiar do relógio.** As falhas começaram exatamente na janela em que os comentários
  foram reescritos.
- **`ARJUNA016053` nunca diz o que é.** Antes de caçar no servidor (lock, instância em
  andamento, ambiente), conferir o que **mudou no projeto** desde o último publish OK.
- Teste que separa as duas famílias de causa em 2 minutos: **republicar o HTML antigo que já
  subiu**. Se funcionar, o problema é o conteúdo novo; se falhar igual, é estado do servidor.
