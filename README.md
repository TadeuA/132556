# Relatório de Problemas Identificados

## Rotina de Correção de Prazos de Relatórios Prorrogados

**Rotina:** `corrigirPrazoRelatorioProcessosProrrogados`

**Contexto:** A rotina processa relatórios finais (tipo RMMF) que estão com data de término real do processo superior à data de término de referência do relatório, e que estão em situação aguardando envio (APE, REC, ATRC, ATRB).

---

## Problemas Críticos

### 1. Projetos curtos podem gerar relatórios com período impossível

#### O que acontece hoje

Quando um processo tem a configuração de "meia execução" (dividir em dois relatórios), o sistema usa uma fórmula fixa que pode resultar em períodos negativos ou zero para o primeiro relatório.

**Trecho do código:**

```php
// Calcula duração do primeiro relatório
$duracaoPrimeiroRelatorio = (intdiv($duracaoMeses, 2) - 2);

// Exemplo: Processo de 3 meses
// Resultado: (3 ÷ 2) - 2 = 1 - 2 = -1 mês
```

#### Por que isso é um problema

- **Datas de início ficam maiores que datas de término** (impossível)
- **Sistema pode falhar e gerar erros** ao tentar processar essas datas
- **Pesquisador recebe prazo impossível de cumprir** (relatório de período zero ou negativo)

#### Exemplos práticos

| Duração do Processo | Cálculo            | Duração 1º Relatório | Situação     |
| ------------------- | ------------------ | -------------------- | ------------ |
| 6 meses             | (6÷2)-2 = 1 mês    | 1 mês                | Funciona     |
| 4 meses             | (4÷2)-2 = 0 meses  | 0 meses              | Período zero |
| 3 meses             | (3÷2)-2 = -0,5 = 0 | 0 meses              | Período zero |
| 2 meses             | (2÷2)-2 = -1 mês   | Negativo             | Impossível   |

#### Impacto no dia a dia

```
Processo: Bolsa de Iniciação Científica - 3 meses
Configuração: Dividir em 2 relatórios (meia execução)

Sistema tenta criar:
  Relatório 1: 01/01 a 01/01 (0 dias)
  Relatório 2: 02/01 a 31/03 (todo o período)
```

#### Dúvidas

1. **Qual deve ser a duração mínima aceitável para um relatório parcial?**
2. **O que fazer quando o projeto é curto demais para dividir em dois relatórios?**
3. **Existe regra de negócio que defina quando aplicar ou não a divisão em dois relatórios?**

---

## Problemas de Média Gravidade

### 2. Cálculo de duração pode não corresponder ao esperado em meses

#### O que acontece hoje

O sistema usa duas formas diferentes de trabalhar com "meses":

1. **Para adicionar meses a uma data** - Usa função nativa PHP/SQL que considera meses de calendário
2. **Para calcular quantos meses** - Usa função customizada que conta meses de calendário
3. **Para calcular duração do processo** - Divide dias por 30 (média fixa)

**Trechos do código:**

```php
// Forma 1: Adicionar meses (usa calendário)
$dataFinal = date("Y-m-d", strtotime($dataInicio . "+" . $meses . "MONTH"));
// 01/01 + 6 MONTH = 01/07 (considera calendário)

// Forma 2: Calcular meses entre datas (usa função customizada)
$quantidadeMeses = Funcoes::calcularDiferencaEntreDatas(
    $relatorio->dataInicioReferencia,
    $dataTerminoReal,
    'mes'
);
// Conta meses de calendário (Janeiro = 1, Fevereiro = 2, etc)

// Forma 3: Calcular duração em meses (usa divisão por 30)
$diferenca = strtotime($dataFim) - strtotime($dataInicio);
$duracaoMeses = intdiv($diferenca, (30 * 60 * 60 * 24));
// 181 dias ÷ 30 = 6 meses + 1 dia
```

#### Como a função calcularDiferencaEntreDatas funciona

A função conta quantos meses de calendário existem entre duas datas:

```php
// Exemplo: 15/01/2025 a 20/03/2025
// Conta: Janeiro, Fevereiro, Março = 3 meses
// Depois subtrai 1 = 2 meses

// Exemplo: 01/01/2025 a 31/03/2025
// Conta: Janeiro, Fevereiro, Março = 3 meses
// Depois subtrai 1 = 2 meses
```

**Observação:** Esta função retorna quantidade de meses cheios, não considera dias parciais.

#### Por que isso pode gerar diferenças

**Na maioria dos casos funciona bem**, mas podem haver diferenças dependendo do método usado:

```
Exemplo 1:
  Período: 31/01/2025 a 28/02/2025

  Método calendário (calcularDiferencaEntreDatas):
    Janeiro, Fevereiro = 2 meses - 1 = 1 mês

  Método divisão por 30:
    28 dias ÷ 30 = 0 meses + 28 dias

Exemplo 2:
  Período: 01/01/2025 a 01/07/2025

  Método calendário (calcularDiferencaEntreDatas):
    Jan, Fev, Mar, Abr, Mai, Jun, Jul = 7 - 1 = 6 meses

  Método divisão por 30:
    181 dias ÷ 30 = 6 meses + 1 dia

  Neste caso coincide!

Exemplo 3:
  Período: 15/01/2025 a 20/02/2025

  Método calendário (calcularDiferencaEntreDatas):
    Jan, Fev = 2 - 1 = 1 mês

  Método divisão por 30:
    36 dias ÷ 30 = 1 mês + 6 dias

  Também coincide!
```

#### Impacto nas decisões do sistema

Este cálculo é usado para:

- Decidir se cria novo relatório ou apenas estende o atual
- Calcular se ultrapassou o limite de meses configurado no edital
- Definir quando dividir entre parcial e final

**Consequência potencial:**

```
Dependendo do método usado, dois processos com mesma duração
podem ter tratamentos diferentes.
```

#### Dúvidas

1. **Quando um edital define "relatórios a cada X meses", isso significa X meses de calendário (janeiro, fevereiro, março...) ou X × 30 dias corridos**

2. **Essa diferença já causou problemas ou questionamentos?**

---

### 3. Sistema pode criar relatórios SEM número (risco baixo, nunca aconteceu)

#### O que acontece hoje

Quando o sistema tenta gerar um número para o novo relatório, se ocorrer alguma falha, o erro não é tratado adequadamente.

**Trecho do código:**

```php
try {
    $retorno = DB::select('EXEC ['.env('DB_DATABASE_EVEREST')."].dbo.GERARID 'RMM'");
    return $retorno[0]->CHAVE;
} catch (\TroubleException $e) {
    // Não faz nada quando dá erro!
    // Não lança exceção, não loga, não retorna valor
}
```

**Importante:** Este problema **nunca foi reportado** em produção até o momento, mas é uma situação que **pode acontecer** em casos raros de:

- Indisponibilidade momentânea do banco de dados
- Falha na procedure GERARID

Se acontecer, relatórios sem número não podem ser rastreados adequadamente e documentos oficiais não podem ser gerados.

#### Dúvidas

1. **Considerando que nunca aconteceu, vale a pena implementar tratamento?**

2. **Se acontecer, o que deve ser feito?**

---

### 4. Sistema não valida se quantidade de meses configurada é compatível com o projeto

#### O que acontece hoje

O sistema compara a duração real do projeto com a quantidade de meses configurada no edital, mas não valida se a configuração faz sentido no momento do cadastro.

#### Cenários que podem acontecer

**Cenário 1: Configuração maior que o projeto**

```
Projeto: 6 meses de duração
Configuração do edital: Relatórios a cada 12 meses

Resultado:
  - Sistema sempre vai "apenas estender"
  - Nunca vai gerar novo relatório, mesmo com múltiplas prorrogações
```

**Cenário 2: Configuração inconsistente com "meia execução"**

```
Projeto: 6 meses
Configuração: Meia execução = SIM + Relatórios a cada 3 meses

Conflito potencial:
  - "Meia execução" deveria gerar 2 relatórios de 3 meses cada
  - Mas "a cada 3 meses" também geraria 2 relatórios
  - Qual regra prevalece em caso de prorrogação?
```

#### Impacto

- **Comportamento pode ser imprevisível** dependendo da configuração
- **Erro só é percebido quando processos são prorrogados**

#### Dúvidas

1. **Deve haver validação entre duração do projeto e periodicidade dos relatórios?**

2. **O que prevalece em caso de conflito de configurações?**

3. **Essa situação já aconteceu e causou problemas?**

---

## Problemas de Menor Impacto

### 5. Sistema pode gerar mais de 99.999 relatórios em um ano?

#### O que acontece hoje

O sistema de numeração está preparado para gerar até 99.999 relatórios por ano (formato: RMM-00001-25 até RMM-99999-25).

**Trecho do código SQL:**

```sql
-- Calcula quantos zeros colocar antes do número
SELECT @QTDCARACTERES = (5 - LEN(@VALORSTR))

-- Se o número precisar de mais de 5 dígitos
IF @QTDCARACTERES < 0
BEGIN
  SELECT NULL  -- Retorna nada
END
```

#### Por que isso provavelmente não é um problema

- **Volume atual não chega perto** de 100.000 relatórios/ano
- **Crescimento esperado não indica risco** em curto/médio prazo
- **Seria necessário crescimento** de mais de 270 relatórios por dia útil

**Mas é importante saber que existe:**
Se algum dia chegasse a 99.999, o próximo retornaria NULL e seria criado sem número.

---
