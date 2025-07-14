# Rotina Atualizar Instituições Inadimplentes

A função `atualizarInstituicoesInadimplentes` pode ser acionada atravez de uma rotina automática do sistema ou manualmente pela plataforma Evando Mira. Ela identifica e marca instituições como inadimplentes com base em certidões vencidas.

###### Regra de Negócio
Uma instituição torna-se inadimplente quando:
- Possui certidões com data de vencimento anterior à data atual
- Ainda não está marcada como inadimplente (`INADIMPLENTE = 0`)
- Está ativa no sistema (`INATIVO = 0` ou `NULL`)

## 1. Problemas da Implementação Original

### 1.1. **Ineficiência de Performance**
```php
//  Implementação original - 3 consultas desnecessárias
$quantidade = ViewRecuperarInstituicoesInadimplentesIn::
	recuperarQuantidadeInstituicoesInadimplentesIn();
if ($quantidade > 0) {
    DB::statement('EXEC ['.env('DB_DATABASE_EVEREST').'].dbo.INSTITUICOESINADIMPLENTESIN');
}
```

**Problemas:**
- Executava a **mesma consulta 3 vezes** (1x na view + 2x na procedure)
- Overhead desnecessário de rede e processamento
- Consulta redundante apenas para contar registros

### 1.2. **Transações Aninhadas Desnecessárias**
```php
//  Transação dupla
DB::beginTransaction();
// A procedure também tinha BEGIN TRAN ... COMMIT
DB::statement('EXEC ...');
DB::commit();
```

**Problemas:**
- Redundância de controle transacional
- Complexidade adicional no gerenciamento de erros
- Overhead de performance

### 1.3. **Falta de Dados Precisos**
A implementação original não retornava a quantidade exata de registros processados, apenas uma estimativa baseada na contagem prévia.


### 1.4. **Procedure retornando múltiplos conjuntos de resultados **
**Problemas:**
- SQL Server retorna **múltiplos conjuntos de resultados** para comandos como INSERT/UPDATE
- Cada comando DML retorna uma mensagem de "X linhas afetadas"
- PHP Laravel fica confuso com múltiplas respostas
- `DB::statement()` não consegue completar toda a procedure e termina antes do esperado

## 2. Melhorias Implementadas

### 2.1. **Eliminação da Consulta Redundante**
```php
// Implementação otimizada - 1 consulta apenas
$resultado = DB::select('EXEC ['.env('DB_DATABASE_EVEREST').'].dbo.INSTITUICOESINADIMPLENTESIN');
$quantidadeRegistrosAfetados = $resultado[0]->QuantidadeAfetada ?? 0;
```

### 2.2. **Controle Transacional Único**
A procedure agora é a unica gerenciar a transação, eliminando a duplicidade:
```sql
-- Na procedure
BEGIN TRY
    SET NOCOUNT ON;
    BEGIN TRAN
    -- lógica de negócio
    COMMIT
END TRY
```

### 2.3. **Retorno de Dados Precisos**
A procedure agora retorna a quantidade exata de registros processados:
```sql
SET @QuantidadeAfetada = @@ROWCOUNT
SELECT @QuantidadeAfetada AS QuantidadeAfetada
```

### 2.4. **Refatoração do Código**
Extração da lógica de logging para método separado (`registrarExecucaoRotina`), melhorando a organização e reutilização.

### 2.5. Adição do SET NOCOUNT ON
```sql
SET NOCOUNT ON;
```
O `SET NOCOUNT ON` suprime as mensagens de "X linhas afetadas", garante que apenas o SELECT final seja retornado e permite que o PHP receba corretamente o resultado da procedure.

## 3. Passo a Passo do Código Atual

### 3.1. **Preparação**
```php
$usuarioId = $request->header('usuarioId');
$rotinaSistema = RotinaSistema::recuperarRotinaSistemaAtivaPelaChave('AIIN');
$dataInicial = Funcoes::retornarDataFormatoBancoDados();
```
- Captura dados da requisição
- Localiza a configuração da rotina no sistema
- Registra timestamp de início

### 3.2. **Inicialização de Variáveis**
```php
$quantidadeRegistrosAfetados = 0;
$logErro = null;
$sucesso = 0;
```
- Prepara variáveis de controle para logging

### 3.3. **Validação da Rotina**
```php
if ($rotinaSistema != null) {
```
- Verifica se a rotina está cadastrada e ativa no sistema

### 3.4. **Execução Principal**
```php
try {
    $resultado = DB::select('EXEC ['.env('DB_DATABASE_EVEREST').'].dbo.INSTITUICOESINADIMPLENTESIN');
    $quantidadeRegistrosAfetados = $resultado[0]->QuantidadeAfetada ?? 0;
    
    if ($quantidadeRegistrosAfetados == 0) {
        $logErro = 'Nenhuma instituição encontrada ou processada';
    }
    
    $sucesso = 1;
} catch (\Exception $e) {
    $quantidadeRegistrosAfetados = 0;
    $logErro = $e->getMessage();
}
```
- Executa a procedure otimizada
- Captura o resultado real de registros processados
- Trata erros adequadamente

#### 3.4.1 **Procedure INSTITUICOESINADIMPLENTESIN**
Esta procedure identifica e marca instituições como inadimplentes baseado em certidões com data de vencimento vencida.
#### 3.4.1.1. Inicialização`
```sql
BEGIN TRY
	SET NOCOUNT ON;
    BEGIN TRAN
    DECLARE @QuantidadeAfetada INT = 0
```
- Define `SET NOCOUNT ON` para otimizar performance
- Inicia uma transação (`BEGIN TRAN`)
- Declara variável `@QuantidadeAfetada` para controlar quantos registros foram processados

#### 3.4.1.2. Inserção no Histórico
```sql   
    SET DATEFORMAT DMY
    INSERT INTO INSTITUICOESHISTORICO (IDINSTITUICAO, INADIMPLENTE, USUARIOCRIACAO, DATACRIACAO)
    SELECT DISTINCT(C.IDINSTITUICAO) AS IDINSTITUICAO, 1, 'SA', GETDATE()
    FROM CERTIDOES C (NOLOCK)
    WHERE DATEDIFF(DAY, C.DATAVENCIMENTO, GETDATE()) > 0
      AND EXISTS (SELECT 1 FROM INSTITUICOES TMP (NOLOCK) 
                  WHERE TMP.IDINSTITUICAO = C.IDINSTITUICAO 
                    AND TMP.INADIMPLENTE = 0 
                    AND (TMP.INATIVO = 0 OR TMP.INATIVO IS NULL))
```
- Insere registros na tabela `INSTITUICOESHISTORICO` para cada instituição que deve ser marcada como inadimplente
- **Critérios de seleção:**
  - Certidões com `DATAVENCIMENTO` menor que a data atual
  - Instituições que estão atualmente adimplentes (`INADIMPLENTE = 0`)
  - Instituições ativas (`INATIVO = 0` ou `INATIVO IS NULL`)

#### 3.4.1.3. Atualização do Status
  ```sql
   SET DATEFORMAT DMY
    UPDATE INSTITUICOES
    SET INADIMPLENTE = 1
    WHERE IDINSTITUICAO IN (
        SELECT DISTINCT(C.IDINSTITUICAO) AS IDINSTITUICAO 
        FROM CERTIDOES C (NOLOCK)
        WHERE DATEDIFF(DAY, C.DATAVENCIMENTO, GETDATE()) > 0
          AND EXISTS (SELECT 1 FROM INSTITUICOES TMP (NOLOCK) 
                      WHERE TMP.IDINSTITUICAO = C.IDINSTITUICAO 
                        AND TMP.INADIMPLENTE = 0 
                        AND (TMP.INATIVO = 0 OR TMP.INATIVO IS NULL))
    )
  ```
- Atualiza o campo `INADIMPLENTE` para `1` nas instituições identificadas
- Usa os mesmos critérios do passo anterior

#### 3.4.1.4. Obtem a quantidade de registros afetados
```sql
    SET @QuantidadeAfetada = @@ROWCOUNT
```
- Captura a quantidade de registros afetados com `@@ROWCOUNT`

#### 3.4.1.5. Finalização
```sql
 COMMIT
    SELECT @QuantidadeAfetada AS QuantidadeAfetada
```
- Confirma a transação (`COMMIT`)
- Retorna a quantidade de registros processados

#### 3.4.1.6. Tratamento de Erro
```sql
    IF @@TRANCOUNT > 0 
        ROLLBACK
    SELECT 0 AS QuantidadeAfetada, ERROR_MESSAGE() AS MensagemErro
```
- Em caso de erro, executa `ROLLBACK` da transação
- Retorna `0` como quantidade afetada
- Retorna descrição do erro

### 3.5. **Logging da Execução**
```php
self::registrarExecucaoRotina(
    $rotinaSistema->id, 
    $dataInicial, 
    $dataFinal, 
    $usuarioId, 
    $quantidadeRegistrosAfetados, 
    $sucesso, 
    $logErro
);
```
- Registra a execução para auditoria e monitoramento
- Inclui dados precisos sobre o processamento

### 3.6. **Resposta ao Cliente**
```php
return response()->json([
    'mensagem' => $mensagem->mensagem,
    'quantidadeProcessada' => $quantidadeRegistrosAfetados
], $mensagem->codigo);
```
- Retorna status da execução
- Inclui quantidade precisa de registros processados

## 4. Queries de Verificação de Sucesso

### 4.1. **Query ANTES de executar a rotina**
- Verificar quantas instituições serão afetadas
```sql
SELECT COUNT(DISTINCT C.IDINSTITUICAO) AS 'Instituições que serão marcadas como inadimplentes'
FROM CERTIDOES C (NOLOCK)
WHERE DATEDIFF(DAY, C.DATAVENCIMENTO, GETDATE()) > 0
  AND EXISTS (SELECT 1 FROM INSTITUICOES TMP (NOLOCK) 
              WHERE TMP.IDINSTITUICAO = C.IDINSTITUICAO 
                AND TMP.INADIMPLENTE = 0 
                AND (TMP.INATIVO = 0 OR TMP.INATIVO IS NULL));
```
- Listar as instituições que serão afetadas (com detalhes)
```sql
SELECT DISTINCT 
    I.IDINSTITUICAO,
    I.NOME,
    I.INADIMPLENTE AS 'Status Atual',
    COUNT(C.IDCERTIDAO) AS 'Qtd Certidões Vencidas'
FROM INSTITUICOES I (NOLOCK)
INNER JOIN CERTIDOES C (NOLOCK) ON I.IDINSTITUICAO = C.IDINSTITUICAO
WHERE DATEDIFF(DAY, C.DATAVENCIMENTO, GETDATE()) > 0
  AND I.INADIMPLENTE = 0 
  AND (I.INATIVO = 0 OR I.INATIVO IS NULL)
GROUP BY I.IDINSTITUICAO, I.NOME, I.INADIMPLENTE
ORDER BY I.NOME;
```

- Verificar status atual das instituições
```sql
SELECT 
    COUNT(*) AS 'Total Instituições',
    SUM(CASE WHEN INADIMPLENTE = 1 THEN 1 ELSE 0 END) AS 'Inadimplentes',
    SUM(CASE WHEN INADIMPLENTE = 0 THEN 1 ELSE 0 END) AS 'Adimplentes'
FROM INSTITUICOES (NOLOCK)
WHERE (INATIVO = 0 OR INATIVO IS NULL);
```

### 3.4.2. **Query DEPOIS de executar a procedure**
- Verificar quantas instituições foram processadas
```sql
SELECT 
    COUNT(*) AS 'Total Instituições',
    SUM(CASE WHEN INADIMPLENTE = 1 THEN 1 ELSE 0 END) AS 'Inadimplentes',
    SUM(CASE WHEN INADIMPLENTE = 0 THEN 1 ELSE 0 END) AS 'Adimplentes'
FROM INSTITUICOES (NOLOCK)
WHERE (INATIVO = 0 OR INATIVO IS NULL);
```
- Verificar os registros inseridos no histórico (últimos inseridos)
```sql
SELECT TOP 10
    IH.IDINSTITUICAO,
    I.NOME,
    IH.INADIMPLENTE,
    IH.USUARIOCRIACAO,
    IH.DATACRIACAO
FROM INSTITUICOESHISTORICO IH (NOLOCK)
INNER JOIN INSTITUICOES I (NOLOCK) ON IH.IDINSTITUICAO = I.IDINSTITUICAO
WHERE IH.USUARIOCRIACAO = 'SA'
ORDER BY IH.DATACRIACAO DESC;
```
- Verificar se não existem mais instituições adimplentes com certidões vencidas
```sql
SELECT COUNT(DISTINCT C.IDINSTITUICAO) AS 'Instituições adimplentes com certidões vencidas (deve ser 0)'
FROM CERTIDOES C (NOLOCK)
WHERE DATEDIFF(DAY, C.DATAVENCIMENTO, GETDATE()) > 0
  AND EXISTS (SELECT 1 FROM INSTITUICOES TMP (NOLOCK) 
              WHERE TMP.IDINSTITUICAO = C.IDINSTITUICAO 
                AND TMP.INADIMPLENTE = 0 
                AND (TMP.INATIVO = 0 OR TMP.INATIVO IS NULL));
```
