# SQL Server 

### **Conceitos**

**Tipos de Tabelas**
- **Tabela Heap**
    - Tabela sem índice clustered.
    - As páginas estão espalhadas pelo disco sem qualquer organização e sem estrutura intermediária de organização e de acesso rápido.
    - Toda vez que solicitamos algum dado é necessário realizar varredura, registro a registro, até encontrar o dado.
- **Tabela Clustered**
    - Tabela com índice clustered. 
    - As páginas de dados são organizadas e a tabela possui estruturas intermediárias de organização e de acesso rápido.
    - A tabela fica nos nós folhas da árvore.

**Tipos de Índices** 
- **Índice Clustered**
    - Estrutura intermediária de organização e de acesso rápido à tabela que interfere diretamente na organização das páginas de dados.
    - Quando uma tabela possui índice clustered a modificação de campos chaves (inclusão, alteração e exclusão) dos registros exige a reorganização das páginas de dados para se garantir uma ordenação.
    - Toda tabela só pode ter um índice clustered.
    - Por padrão o SQL Server cria o índice clustered automaticamente sempre que indicamos uma chave primária, porém é possível definir o índice clustered para qualquer combinação de campos da tabela, para isso é necessário excluir o índice e criar novamente.
    - A existência do índice clustered não garante que as páginas estão fisicamente ordenadas. A existência do índice clustered significa que existe um esforço para tentar manter uma certa organização, mas a ordenação física nem sempre é garantida. Entretanto a ordenação lógica é certeza.
- **Índice Nonclustered** 
    - Estrutura intermediária de organização e de acesso rápido à tabela que não interfere na organização das páginas de dados.
    - Pode-se definir centenas de índices nonclustered por tabela.
    
**Índice Nonclustered com Tabela Clustered** 

Quando temos um índice nonclustered sobre uma tabela clustered, ou seja, sobre uma tabela com índice clustered, o índice nonclustered armazena as colunas do índice nonclustered e também as colunas do índice clustered. Se o sql server decicir que a busca deve passar pelo índice nonclustered e este não possuir todas as colunas necessárias o sql server vai passar também pelo índice clustered até chegar no nó folha. Esse processo de busca que passa pelo índice nonclustered e depois pelo índice clustered se chama **Key Lookup**.

**Índice Nonclustered com Tabela Heap** 

Quando temos um índice nonclustered sobre uma tabela heap, ou seja, sobre uma tabela que não possui índice clustered, o índice nonclustered armazena as colunas do índice nonclustered e um ponteiro  (nro_do_arquivo:página:posição_do_registro_na_página) que aponta diretamente para o registro do dado na tabela heap. Se o sql server decidir que a busca deve passar pelo índice nonclustered e este não possuir todas as colunas necessárias o sql server vai ter que acessar também os registros na tabela heap. Esse processo de busca que passa pelo índice nonclustered e depois acessa diretamente o registro na heap se chama **RID Lookup**.

> Teoricamente fica claro que **índice nonclustered com tabela heap** é melhor que **índice nonclustered com tabela clustered** quando se trata de operações de leitura, porém a definição de índice clustered nos traz muitos benefícios e por isso dificilmente criaremos uma tabela heap.

**Chave Primária x Índice Clustered** 

A chave primária é uma restrição, já o índice é uma estrutura auxiliar intermediária de organização e acesso rápido.

**Page Split**   
O SQL Server armazena os dados em páginas (blocos) de 8KB de tamanho fixo. Isso significa que o SQL Server vai acomodando os registros numa página até que a página não suporte novos registros, sendo necessário então a alocação de uma nova página de 8KB. Além de alocar uma nova página de 8KB o SQL Server move metade (50%) dos dados de uma página cheia para a nova página. Esse processo se chama **page split**.

**Fragmentação Lógica ou Fragmentação Externa**  
Fragmentação são **espaços/buracos** ociosos que vão surgindo nas páginas por causa do **page split**. Esses espaços podem surgir nas páginas dos índices e também nas páginas dos dados. Para diminuir a fragmentação de uma tabela (páginas de índices e páginas de dados) podemos definir um **índice clustered** baseado em colunas/campos que favorecem a ordenação. Ainda que utilizamos índice clustered devidamente ordenado se ocorrer exclusões pode surgir fragmentação. Para evitar 100% de fragmentação basta evitar a exclusão física.

Podemos também evitar page split através dos parâmetros de configuração **Fill Factor** e **Pad Index**. Estes parâmetros forçam o banco de dados a deixar espaço disponível nas páginas. 

**Densidade e Seletividade**  
- **Densidade**
    - Quando uma coluna possui alto índice de repetição isso significa que a coluna apresenta alta densidade. Exemplo de colunas que apresentam alta densidade: Sexo, Estado Civil, Cidade, Estado.
- **Seletividade** 
    - Quando uma coluna possui baixo índice de repetição isso significa que a coluna apresenta alta seletividade. Exemplo de colunas que apresentam alta seletividade: ID, Código, CPF, CNPJ.

> Esses dois conceitos são fundamentais para o banco de dados decidir se deve usar um índice (SEEK) ou se deve realizar uma varredura (SCAN) para resvolver uma consulta.

- _Quanto maior a densidade ou menor a seletividade_, menor a probabilidade do banco usar um índice nonclustered.
- _Quanto menor a densidade ou maior a seletividade_, maior a probabilidade do banco usar um índice nonclustered.
- _Quando os dados de pesquisa estão no índice clustered_ muito provavelmente o banco usará o índice clustered.
- _Quanto mais seletivo for o predicado (where [coluna][condição][valor])_, maior é a probabilidade do banco utilizar um índice.

**Plan Cache** 

Área de armazenamento de planos de execução para reutilização e melhoria de performance. A definição de qual plano de execução deve ser usado é uma tarefa custosa e por isso o SQL Server salva os planos em cache para reutiliza-los.

Para listar os planos em cache:  
**SELECT * FROM sys.dm_exec_cached_plans CROSS APPLY sys.dm_exec_sql_text(plan_handle) WHERE usecounts>1 ORDER BY usecounts DESC;**

> É possível remover um plano do cache atravé do comando **DBCC FREEPROCCACHE ...**

**Processos do Plano de Execução**   

- Processos que ocorrem na **Relational Engine**:  
    - **Parse ou Query Parsing**: verifica sintaxe.
    - **Algebrizer**: verifica se já existe plano otimizado em cache e se não existir gera um plano inicial em formato de árvore (plano ainda em nível de abstração).
    - **Query Optimizer**: se o plano otimizado ainda não existe em cache realiza uma série passos de otiminização até que se chegue num plano melhor otimizado ou até que se chegue num limite de tempo (timeout). No final armazena o melhor plano otimizado (plano atual) no cache.
    - _No final temos o Plano de Execução gerado em formato binário e armazenado em cache._
- Processos que ocorrem na **Storage Engine**:  
    - **Query Execution**: recebe o plano de execução otimizado em formato binário, executa o plano e retorna os dados. Durante a execução primeiro verifica se os dados estão em cache/memória, se não estiver em memória acessa o disco e salva em cache/memória.

--- 

### **Comandos de Administração** 

- **DBCC CHECKDB**: verifica a integridade do banco de dados e realiza pequenas correções. 
- **DBCC CHECKTABLE**: verifica a integridade de uma tabela e realiza pequenas correções. 
- **EXEC SP_HELPINDEX 'tabela'**: lista os índices de uma tabela.
- **SELECT * FROM sys.indexes WHERE object_id = object_id('tabela')**: lista os índices de uma tabela.
- **DBCC REINDEX**: reorganiza índices.
- **DROP INDEX nome_do_indice ON nome_da_tabela**: exclui um índice.
- **ALTER TABLE nome_tabela DROP CONSTRAINT nome_pk_ou_fk**: exclui uma constraint.
- **CREATE NONCLUSTERED INDEX nome_do_indice ON tabela (campo1, campo2, ...)**: cria um índice nonclustered.
- **SELECT * FROM tabel WITH (index = indice) WHERE coluna = valor**: força o uso de um índice.
- **DBCC REBUILD**: reorganiza páginas de dados.
- **DBCC SHOW_STATISTICS('nome_tabela','nome_indice')**: exibe as estatísticas de um índice.
- **UPDATE STATISTICS nome_tabela nome_indice**: atualiza as estatísticas de um índice.
- **EXEC SP_UPDATESTATS 'resample'**: atualiza as estatísticas do banco inteiro.
- **DBCC SHOWCONTIG**: exibe uma série de informações sobre as páginas das tabelas. Pode ser usado para verificar o percentual de fragmentação das páginas. 
- **DBCC INDEXDEFRAG**: desfragmenta índice. Pode ser usado com o banco online. Não libera espaço, para isso é necessário fazer **shrink**.
- **EXEC SP_SPACEUSED 'tabela'**: exibe informações sobre o espaço usado e sobre o tamanho de um registro.
- **DBCC IND('base_de_dados','tabela','id_do_indice)**: lista as páginas de um índice. 
    - Cada índice de uma tabela possui um ID (número de identificação do índice). No caso de índice clustered, o ID é 1. Portanto para verificar as páginas de um índice clustered basta informa no comando id_do_indice=1.
    - As páginas com PageType=10 são páginas especiais, são páginas do tipo IAM (Indice Allocation Map), ou seja, é um mapa de páginas que armazena as páginas alocadas e não alocadas do índice.
    - A página com PageType=2, PrevPageFID=0 e NextPageFID=0 é a página root.
    - As demais páginas com PageType=2 são as páginas intermediárias.
- **Para detalhar o conteúdo de uma página**
    1. **DBCC TRACEON(3604,-1)**: sinaliza para o banco de dados que realizaremos uma leitura de página.
    2. **DBCC PAGE('base_de_dados',id_do_indice,nro_da_pagina,3) WITH TABLERESULTS** ou apenas **DBCC PAGE('base_de_dados',id_do_indice,nro_da_pagina,3)**: exibe a página e seus detalhes.
        - O resultado do comando nos traz:  
            - m_prevPage: página anterior.
            - m_nextPage: próxima página. 
            - m_slotCnt: quantidade de registros usados. 
            - m_freeCnt: quantidade de registros livres. 
            - m_freeData: quantidade de registros livres em bytes. 
            - Slot 0: primeiro registro
            - Slot 1: segundo registro
            - Slot 2: terceiro registro
            - ...
            - Slot N: enésio registro.
        - Para descobrir a página e o slot a partir da chave primária:
            - select sys.fn_physlocformatter(%%physloc%%) from [tabela] where [coluna_pk] = [valor]
- **Para habilitar a verificação de estatísticas de IO durante uma consulta**
    - SET STATISTICS IO ON
    - select ...
    - SET STATISTICS IO OFF

--- 

### **Operadores de um Plano de Execução** 

- **Table Scan**: varredura em página de dados, significa que o optimizer decidiu não passar por índice e sim passar registro por registro de dados até encontrar os dados solicitados.
- **Index Scan ou Clustered Index Scan**: varredura em página de índice, significa que o optimizer decidiu passar por todas as linhas de algum índice que pode ser clustered ou nonclustered. Em algumas situações o optimizer pode decidir varrer o índice inteiro e filtrar em vez de simplesmente utilizar as chaves anexadas ao índice.
- **Index Seek**: não foi necessário fazer qualquer tipo de varredura, ou seja, o optimizer entendeu que passar pelo índice (que pode ser nonclustered ou clustered) e usar as chaves anexadas é suficiente e ao mesmo tempo a melhor escolha para se chegar nos dados solicitados. 
- **Key Lookup ou Bookmark Lookup**: significa que decidiu-se passar por um índice nonclustered, mas foi necessário passar pelo índice clustered até chegar no nó folha.
- **RID Lookup**: significa que decidiu-se passar por um índice nonclustered, mas foi necessário acessar a tabela heap. 

--- 

### **Boas Práticas** 

- Defina chaves primárias com colunas que permitam ordenação lógica e física.
- Realize consultas mais seletivas.
- Evite a definição de predicados dinamicamente.
- Não utilize o asteristico (**\***) nas consultas, pois isso exige a expensão das colunas num passo interno. Seja explicito e informe apenas as colunas necessárias para o problema em questão.
- Evite a utilização de funções de usuário, principalmente funções de formatação. Deixe a formatação para a aplicação.
- Separe campos binários/blobs dos campos principais da tabela.
- Evite a exclusão física em tabelas clustereds, pois a exclusão gera espaços/buracos que exigem page split.
- Atualize as estatísticas de tempo em tempo, pois estatísticas defasadas afetam o desempenho.
- Reorganize os índices de tempo em tempo.
- Reorganize as páginas de dados de tempo em tempo.
- Revise os índices nonclustereds já existentes de tempo em tempo.
- Deixe explícito para o banco de dados se o índice nonclustered é único ou não.
- Evite subsconsultas e dê preferência, sempre que possível, para junção.
- Trate a fragmentação de todas as bases de dados de tempo em tempo, inclusive das bases do sistema.
- Verifique a consistência de todos as bases de dados de tempo em tempo, inclusive das bases do sistema.
- Realize backup periódico de todas as bases de dados, inclusive das bases do sistema.

--- 

**Fontes** 

- https://www.youtube.com/user/gmasql (Gustavo Maia)
- https://www.youtube.com/user/mcflyamorim (Fabiano Amorin)
- https://www.youtube.com/channel/UC8EUZ3gYTxJi-gr4azFJGYA (Database Cast) 
- https://www.youtube.com/channel/UCrdqROqmpZ_U6CKt4UEds3g (Esquilo Sequelado) 
- https://www.youtube.com/user/santanche (Professor André Santanchè) 
- https://imasters.com.br/banco-de-dados/sql-server/escovando-bit-com-operadores-key-lookup-e-rid-lookup/ 
- http://www.sqlmagazine.com.br/colunistas/PauloRibeiro/09_Tunning_ExecutionPlan_PT2.asp 
- http://www.devmedia.com.br/sql-server-query-analise-do-plano-de-execucao/30024 
- https://www.devmedia.com.br/tuning-plano-de-execucao-parte-2/2488 
- http://www.linhadecodigo.com.br/artigo/704/sql-server-melhorando-a-performance-atraves-das-estatisticas.aspx 
- http://mcdbabrasil.com.br/ 
- https://www.youtube.com/user/bosontreinamentos (Bóson Treinamentos)