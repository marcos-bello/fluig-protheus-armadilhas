# Compilador Protheus só aceita CP1252

> **Plataforma:** Protheus · AdvPL / TLPP · compilação
> **Status:** confirmado · fonte: TDN

## Sintoma

Fonte `.prw` / `.tlpp` gerado ou editado por ferramenta moderna (VS Code, agente de IA,
qualquer editor com default UTF-8) **não compila** — ou compila e as acentuações viram
*mojibake* (`Ã§`, `Ã£`).

Às vezes o erro aparece como **sintaxe inválida numa linha que está visivelmente correta**, que
é o que faz perder mais tempo.

No SonarQube chega como **CA0000 — erro de compilação, charset errado** (severidade MAJOR).

## Causa

Os compiladores Protheus (AdvPL, AdvPL ASP, 4GL, 4GLP) suportam **apenas Windows-1252
(CP1252)**. Todo editor moderno grava **UTF-8** por padrão.

Em UTF-8 os acentuados ocupam 2 bytes; em CP1252, 1:

| Caractere | UTF-8 | CP1252 |
| --- | --- | --- |
| `é` | `0xC3 0xA9` | `0xE9` |
| `ç` | `0xC3 0xA7` | `0xE7` |
| `ã` | `0xC3 0xA3` | `0xE3` |
| `ü` | `0xC3 0xBC` | `0xFC` |

O compilador lê a sequência errada. E em runtime `Len()` e `SubStr()` passam a contar errado —
`EncodeUTF8()` / `DecodeUTF8()` assumem CP1252 como origem/destino.

> TDN: *"Os compiladores Protheus (AdvPL, AdvPL Asp, 4GL e 4GLP) suportam apenas os arquivos
> com código de página CP1252."*

## Correção

Converter **antes de compilar**, com um script de conversão in-place (`.bat` no Windows, `.sh`
no Linux/macOS — sem dependência externa).

Regras da conversão:

- É **in-place**. O arquivo mantém nome e extensão — `.tlpp` continua `.tlpp`.
- **Nunca** criar `.cp1252`, `.bak`, `.orig`, `.utf8`. Zero cópia, zero artefato — senão você
  compila o arquivo errado uma hora.
- Não converter na mão (copiar e colar entre editores) — corrompe.

Verificar depois:

```bash
file --mime-encoding arquivo.tlpp
# tem que devolver iso-8859-1 ou unknown-8bit — NUNCA utf-8
```

## Como detectar antes

> **Todo fonte que um agente de IA escreveu está em UTF-8.** Não é hipótese, é o default da
> ferramenta.

Regra fixa: **gerou ou editou `.prw` / `.prg` / `.tlpp` / `.prx` → converte → compila.** Nessa
ordem, sempre. Sem exceção para "só mudei uma linha" — uma linha com um `ç` basta.
