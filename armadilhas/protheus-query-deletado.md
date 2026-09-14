# Query sem `D_E_L_E_T_` traz registro apagado

> **Plataforma:** Protheus · SQL · integração e relatório
> **Status:** confirmado · regra de revisão do TOTVS/SonarQube

## Sintoma

Relatório ou integração devolve **mais linhas do que deveria**, ou traz registro que o usuário
jura que apagou.

Nenhum erro. Nenhum log. Só número errado — que é a pior categoria de bug, porque alguém pode
tomar decisão em cima dele antes de qualquer um perceber.

Variante mais traiçoeira: a query tem `D_E_L_E_T_` na tabela principal e **falta em uma das
tabelas do JOIN**. Aí só uma parte do resultado está contaminada, e a conferência por amostra
passa.

## Causa

O Protheus **não apaga registro: marca**. Exclusão é lógica — o campo `D_E_L_E_T_` recebe `'*'`
e a linha continua na tabela.

Pelo mesmo motivo, **falta de filtro de filial** traz dado de todas as filiais — que além de
errado é vazamento entre unidades.

## Correção

Em **toda** tabela de **toda** query:

```sql
WHERE SA1.D_E_L_E_T_ = ' '     -- um espaço, não string vazia
  AND SA1.A1_FILIAL  = '01'
```

Em JOIN, repetir nas duas pontas:

```sql
FROM  SD1010 SD1 WITH (%nolock%)
INNER JOIN SB1010 SB1 WITH (%nolock%)
        ON SB1.B1_FILIAL   = '01'
       AND SB1.B1_COD      = SD1.D1_COD
       AND SB1.D_E_L_E_T_  = ' '      -- <<< esta é a que se esquece
WHERE SD1.D_E_L_E_T_ = ' '
  AND SD1.D1_FILIAL  = '01'
```

Alternativa do framework: a macro DBAccess `%notDel%`.

## Como detectar antes

- **`COUNT(*)` com e sem o filtro.** Diferente = tinha registro apagado entrando.
- Na revisão: **contar quantas tabelas a query tem e quantos `D_E_L_E_T_` aparecem.** Os dois
  números têm que bater. Vale igual para o filtro de filial.
- Nunca hardcodar filial/empresa: `FWxFilial()` e `RetSQLName()`.

> Mesma família dos bugs de **grão** em modelagem de dados: a query "funciona", o número é que
> está errado. Só conferência de cardinalidade e contagem pega — teste de fumaça nunca pega.
