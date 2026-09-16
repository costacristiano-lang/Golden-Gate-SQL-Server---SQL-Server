# OCI GoldenGate: Microsoft SQL Server para Microsoft SQL Server

Guia genérico para configurar replicação SQL Server-to-SQL Server com OCI GoldenGate Data Replication.

> Este documento não depende de nomes, IPs ou compartments específicos. Substitua todos os valores entre `<...>` pelos dados do ambiente.

## Arquitetura

```text
SQL Server origem
        |
        | Extract / CDC
        v
OCI GoldenGate Deployment (Microsoft SQL Server)
        |
        | Distribution Path
        v
SQL Server destino
        |
        v
Replicat
```

Um único Deployment pode conter Extract e Replicat para cenários simples. Para produção, avalie Deployments separados conforme volume, isolamento, disponibilidade e operação.

## 1. Pré-requisitos

Antes de começar, reúna:

- Versão e edição do SQL Server na origem e no destino. Confirme a certificação da versão do Deployment na matriz do OCI GoldenGate.
- Nome DNS/IP, porta TCP e **nome da base de dados** de cada SQL Server. A porta padrão é `1433`, mas use a porta real configurada.
- CIDRs das redes, subnets, NSGs e rotas entre OCI GoldenGate e os servidores SQL Server.
- Um SQL Login dedicado ao GoldenGate, por exemplo `ggadmin`.
- Uma estratégia para a carga inicial e para o ponto de início da captura.
- SQL Server Agent em execução na origem, pois o CDC depende dos jobs do Agent.

Use **SQL Server Authentication** para o usuário do GoldenGate. Não use autenticação Windows/Active Directory para essa connection.

Para servidores SQL Server em VM ou on-premises, o listener deve aceitar conexões TCP remotas na porta configurada. SQL Server em Linux não é suportado como banco SQL Server pelo GoldenGate; valide também a versão/edição e o build do Deployment na matriz oficial.

## 2. IAM

Este roteiro usa o grupo padrão `Administrators`, sem o prefixo do Identity Domain.

Crie ou confirme a policy administrativa:

```text
allow group Administrators to manage all-resources in tenancy
```

O GoldenGate também precisa das policies abaixo para trabalhar com Vault, chaves e IAM Identity Domains:

```text
allow service goldengate to use keys in tenancy
allow service goldengate to use vaults in tenancy
allow service goldengate to {idcs_user_viewer, domain_resources_viewer} in tenancy
```

### Dynamic Group para secrets

Quando connections usam password secrets, o Deployment precisa ler os bundles de secrets.

1. Acesse **Identity & Security → Dynamic Groups → Create Dynamic Group**.
2. Defina um nome, por exemplo `dg-ogg-deployments`.
3. Use a regra abaixo, substituindo o OCID do compartment do Deployment:

```text
ALL {resource.type = 'goldengatedeployment', resource.compartment.id = '<compartment-ocid>'}
```

4. Crie a policy:

```text
allow dynamic-group dg-ogg-deployments to read secret-bundles in tenancy
```

## 3. Rede

Crie ou reserve uma subnet **privada**, sem sobreposição com outras subnets e contida no CIDR da VCN. Exemplo:

```text
VCN:                  10.0.0.0/16
Subnet GoldenGate:    10.0.10.0/24
```

Configure NSGs ou Security Lists conforme a topologia:

| Origem | Destino | Protocolo/porta | Finalidade |
|---|---|---|---|
| Bastion, VPN ou rede administrativa | Endpoint GoldenGate | TCP 443 | Console e Admin Client |
| Ingress IPs do GoldenGate | SQL Server origem | TCP 1433 ou porta real | Extract / CDC |
| Ingress IPs do GoldenGate | SQL Server destino | TCP 1433 ou porta real | Replicat |
| Deployment origem | Deployment destino | TCP 443 | Distribution Path remoto |

Para connections com **Shared endpoint**, libere os **Ingress IPs do Deployment**. Para **Dedicated endpoint**, libere os **Ingress IPs exibidos nos detalhes da própria connection**.

Se o SQL Server estiver on-premises ou em outra VCN, valide DRG/LPG, VPN/FastConnect, DNS, firewall do Windows e rota de retorno. O erro `Login timeout expired` normalmente indica rede, porta, listener ou rota — não uma credencial inválida.

## 4. OCI Vault e secrets

### Criar o Vault

1. Acesse **Identity & Security → Vault → Create Vault**.
2. Informe um nome, por exemplo `VAULT-OGG`.
3. Aguarde o estado `Active`.

### Criar a chave

1. Abra o Vault.
2. Acesse **Master Encryption Keys → Create Key**.
3. Crie uma chave AES, por exemplo `KEY-OGG-SECRETS`.

### Criar os secrets

Crie secrets separados para origem e destino:

```text
OGG-SQL-SRC-PASSWORD  → senha do SQL Login ggadmin na origem
OGG-SQL-TGT-PASSWORD  → senha do SQL Login ggadmin no destino
```

Na criação do secret, selecione **Manual secret generation** e informe somente a senha no conteúdo. Ao trocar a senha, crie uma nova versão do secret e use **Refresh connection** na connection do GoldenGate para limpar o valor em cache.

## 5. Preparar o SQL Server origem

Execute os comandos abaixo no SQL Server Management Studio (SSMS) com uma conta administrativa. Substitua `<senha-segura>` e `<database-origem>`.

### Criar o SQL Login e o usuário nas bases

O mesmo login deve existir em `msdb` e na base de origem. `ggadmin` é um **SQL Login**; não é um usuário Windows.

```sql
USE master;
GO
CREATE LOGIN ggadmin WITH PASSWORD = '<senha-segura>', CHECK_POLICY = ON;
GO

USE msdb;
GO
CREATE USER ggadmin FOR LOGIN ggadmin;
ALTER ROLE SQLAgentReaderRole ADD MEMBER ggadmin;
GO

USE [<database-origem>];
GO
CREATE USER ggadmin FOR LOGIN ggadmin;
ALTER ROLE db_owner ADD MEMBER ggadmin;
GO
```

### Habilitar CDC

O Extract de SQL Server usa Change Data Capture (CDC). O DBA pode habilitar o CDC antecipadamente, evitando conceder `sysadmin` ao login do GoldenGate:

```sql
USE [<database-origem>];
GO
EXEC sys.sp_cdc_enable_db;
GO
```

Caso o GoldenGate seja autorizado a habilitar CDC durante `ADD TRANDATA`, o login do Extract precisa de `sysadmin` **temporariamente**. Depois que o CDC estiver habilitado para as tabelas necessárias, remova esse privilégio elevado:

```sql
USE master;
GO
ALTER SERVER ROLE sysadmin ADD MEMBER ggadmin;
GO
-- Execute o ADD TRANDATA no GoldenGate.
ALTER SERVER ROLE sysadmin DROP MEMBER ggadmin;
GO
```

Não deixe `sysadmin` concedido de forma permanente sem necessidade. Para uma base já preparada, `db_owner` na base de origem e `SQLAgentReaderRole` em `msdb` atendem ao fluxo indicado pela Oracle.

### Validar a origem

```sql
USE [<database-origem>];
GO
SELECT name, is_cdc_enabled FROM sys.databases WHERE name = DB_NAME();
SELECT name, is_tracked_by_cdc FROM sys.tables WHERE is_ms_shipped = 0;
GO
```

Confirme que o serviço **SQL Server Agent** está em execução. Para tabelas que participam da replicação, use chave primária ou chave única confiável.

## 6. Preparar o SQL Server destino

No destino, crie o login e o usuário na base de destino. Para Replicat, o usuário também precisa de `db_owner` na base alvo e de `SQLAgentUserRole` em `msdb`.

```sql
USE master;
GO
CREATE LOGIN ggadmin WITH PASSWORD = '<senha-segura>', CHECK_POLICY = ON;
GO

USE msdb;
GO
CREATE USER ggadmin FOR LOGIN ggadmin;
ALTER ROLE SQLAgentUserRole ADD MEMBER ggadmin;
GO

USE [<database-destino>];
GO
CREATE USER ggadmin FOR LOGIN ggadmin;
ALTER ROLE db_owner ADD MEMBER ggadmin;
GO
```

Antes de iniciar o Replicat, confirme que schemas, tabelas, tipos de dados, collation e chaves primárias/únicas sejam compatíveis com a origem. Se a base de destino já tiver dados, defina como a carga inicial será sincronizada antes de iniciar a replicação contínua.

## 7. Criar o Deployment

1. Acesse **Oracle AI Database → GoldenGate → Deployments → Create deployment**.
2. Selecione:

```text
Deployment type:     Data replication
Technology:          Microsoft SQL Server
Version:             build certificado para o SQL Server de origem/destino
Private subnet:      <subnet-golden-gate>
Credential store:    OCI IAM ou GoldenGate
```

3. Aguarde o estado `Active`.
4. Registre a **Console URL**, os **Ingress IPs** e o endereço privado exibidos nos detalhes do Deployment.

Por padrão, o acesso à console é HTTPS na porta `443`. Habilite acesso público somente quando necessário e proteja-o com regras de rede restritivas.

## 8. Criar connections SQL Server

Crie uma connection para a origem em **GoldenGate → Connections → Create connection**:

```text
Name:                          OGG-SQL-SRC
Type:                          Microsoft SQL Server
Database name:                 <database-origem>
Database host:                 <dns-ou-ip-sql-server-origem>
Port:                          1433 ou porta real
Username:                      ggadmin
Database user password secret: OGG-SQL-SRC-PASSWORD
SSL details:                   Plain ou TLS, conforme o SQL Server
Network connectivity:          Shared endpoint ou Dedicated endpoint
```

Repita para o destino:

```text
Name:                          OGG-SQL-TGT
Type:                          Microsoft SQL Server
Database name:                 <database-destino>
Database host:                 <dns-ou-ip-sql-server-destino>
Port:                          1433 ou porta real
Username:                      ggadmin
Database user password secret: OGG-SQL-TGT-PASSWORD
```

O campo **Database name** é o nome da base exibida em **Databases** no SSMS, por exemplo `Newcon_Embracon`; não é o nome do servidor/instância mostrado no topo do Object Explorer.

### DSN ou Host + Port

- **Host + Port:** informe diretamente host/IP, porta e database. É a opção recomendada para OCI GoldenGate, mais simples de operar.
- **DSN:** usa um nome de fonte ODBC previamente configurado, contendo host, porta, driver e outros parâmetros. Use apenas quando a arquitetura exigir um DSN administrado; ele não corrige problema de rede ou de login.

Se usar IP privado, o OCI GoldenGate pode reescrevê-lo em um nome interno no formato `ip-<endereco>.ociggsvc.oracle.vcn.com`; isso é comportamento esperado do serviço.

## 9. Atribuir e testar connections

1. Acesse **GoldenGate → Deployments → `<deployment>` → Assigned connections**.
2. Clique em **Assign connection** e atribua `OGG-SQL-SRC` e `OGG-SQL-TGT`.
3. Nas ações de cada connection, selecione **Test connection**.

O teste deve confirmar dois níveis:

```text
Network-level connectivity: host e porta alcançáveis
Application-level connectivity: SQL Login, senha e database válidos
```

### Diagnóstico de `Login timeout expired`

Esse erro acontece antes da validação de senha. Verifique nesta ordem:

1. O IP/FQDN e a porta da connection estão corretos.
2. TCP/IP está habilitado no SQL Server Configuration Manager.
3. O SQL Server está escutando na porta informada, por exemplo `1433`.
4. O firewall do Windows e firewalls intermediários permitem TCP da origem correta.
5. NSG/Security List permite os Ingress IPs do Deployment ou da connection dedicada.
6. Há rota de ida e retorno entre a subnet do GoldenGate e a rede do SQL Server.

Após a rede responder, se houver erro de credencial, confirme:

```sql
USE master;
GO
SELECT name, type_desc, is_disabled
FROM sys.sql_logins
WHERE name = 'ggadmin';
GO

USE [<database>];
GO
SELECT name, type_desc
FROM sys.database_principals
WHERE name = 'ggadmin';
GO
```

O esperado é `SQL_LOGIN`, `is_disabled = 0` e um usuário `SQL_USER` mapeado na base configurada.

## 10. Configurar a replicação

Abra **Launch console** no Deployment. A ordem geral é:

```text
Extract → Distribution Path → Replicat
```

### Origem

1. Crie um Extract para Microsoft SQL Server.
2. Selecione `OGG-SQL-SRC`.
3. Defina o ponto inicial: hora atual ou posição coordenada com a carga inicial.
4. Habilite `TRANDATA` para os schemas/tabelas necessários; essa etapa configura a captura CDC das tabelas.
5. Configure o trail local.
6. Crie o Distribution Path para o destino.

### Destino

1. Crie um Replicat.
2. Selecione `OGG-SQL-TGT`.
3. Selecione o trail recebido.
4. Defina o mapeamento de schemas e tabelas.
5. Inicie o Replicat.

## 11. Carga inicial

Escolha uma estratégia antes de iniciar a replicação contínua:

- backup/restore ou export/import SQL Server;
- ferramenta de carga externa;
- Extract de carga inicial;
- início coordenado após uma cópia consistente da origem.

Não inicie a captura contínua sem definir como a carga inicial e o ponto de início do Extract serão coordenados. A meta é evitar perda ou duplicação de alterações entre a cópia e o início da replicação.

## 12. Operação e validação

No Admin Client:

```text
INFO ALL
VIEW MESSAGES
```

Valide:

```text
Extract:  RUNNING
Replicat: RUNNING
Lag:      aceitável para o SLA
Erros:    ausentes em VIEW MESSAGES
```

Execute um teste controlado de `INSERT`, `UPDATE` e `DELETE` na origem e confirme o resultado no destino. Registre o horário, a chave do registro e o lag observado.

## 13. Sugestões de desempenho e monitoramento

Use estas sugestões como ponto de partida para ambientes SQL Server. Meça lag, CPU, I/O, uso de memória e tempo de commit antes de elevar valores em produção.

> No OCI GoldenGate, o assistente cria os parâmetros estruturais, como `EXTRACT`, `USERIDALIAS`, trail e checkpoint. Não os duplique no arquivo de parâmetros.

### Extract CDC

Exemplo de parâmetros adicionais para um Extract chamado `ECDC`:

```text
REPORTCOUNT EVERY 5 MINUTES, RATE
REPORTROLLOVER AT 00:05

WARNLONGTRANS 1H, CHECKINTERVAL 10M

TRANLOGOPTIONS TRANCOUNT 20

DISCARDFILE ./dirrpt/ECDC.dsc, APPEND
DISCARDROLLOVER AT 00:10

TABLE dbo.CLIENTE;
TABLE dbo.PEDIDO;
TABLE dbo.ITEM_PEDIDO;
```

| Parâmetro | Finalidade |
|---|---|
| `REPORTCOUNT EVERY 5 MINUTES, RATE` | Registra volume processado, taxa total e taxa do intervalo. |
| `REPORTROLLOVER` | Faz a rotação diária do report para facilitar retenção e análise. |
| `WARNLONGTRANS` | Alerta transações abertas por muito tempo, que podem gerar lag e retenção de CDC. |
| `TRANLOGOPTIONS TRANCOUNT 20` | SQL Server: lê 20 transações CDC por chamada. Comece com `20`; avalie `30` ou `50` somente com base em métricas. |
| `DISCARDFILE` e `DISCARDROLLOVER` | Preservam erros de processamento sem permitir crescimento indefinido do arquivo. |

### Replicat

Exemplo de parâmetros adicionais para um Replicat chamado `RCDC`:

```text
REPORTCOUNT EVERY 5 MINUTES, RATE
REPORTROLLOVER AT 00:15

DISCARDFILE ./dirrpt/RCDC.dsc, APPEND
DISCARDROLLOVER AT 00:20

BATCHSQL
GROUPTRANSOPS 1000

MAP dbo.CLIENTE, TARGET dbo.CLIENTE;
MAP dbo.PEDIDO, TARGET dbo.PEDIDO;
MAP dbo.ITEM_PEDIDO, TARGET dbo.ITEM_PEDIDO;
```

| Parâmetro | Finalidade |
|---|---|
| `BATCHSQL` | Agrupa SQLs semelhantes e tende a aumentar o throughput no destino SQL Server. |
| `GROUPTRANSOPS 1000` | Reduz commits e I/O de checkpoint para lotes de transações pequenas. Não aumente arbitrariamente, pois pode elevar o lag. |
| `REPORTCOUNT` | Permite comparar a taxa de apply com a taxa de captura. |
| `DISCARDFILE` | Registra conflitos, duplicidades, registros ausentes e erros SQL. |
| `MAP` | Define o mapeamento de objetos origem/destino. |

### Cuidados operacionais

- Não mantenha `HANDLECOLLISIONS` em operação contínua; use-o apenas em carga inicial ou cutover controlado, quando aplicável.
- Não configure `REPERROR ... IGNORE` genericamente: isso pode ocultar perda de dados.
- O padrão de `CHECKPOINTSECS` é `10`; não o altere sem evidência de gargalo e validação de impacto em recuperação.
- Em caso de lag no destino, teste `BATCHSQL` antes de ajustar `GROUPTRANSOPS` ou `TRANCOUNT`.
- Configure uma checkpoint table no destino e acompanhe `INFO ALL`, `VIEW MESSAGES`, lag, reports e discard files.

## Troubleshooting rápido

| Sintoma | Causa provável | Ação |
|---|---|---|
| `Login timeout expired` | Listener, porta, NSG, firewall ou rota | Validar TCP até a porta SQL Server e os Ingress IPs |
| `Login failed for user` | SQL Login/senha, usuário mapeado ou login desabilitado | Validar `sys.sql_logins`, usuário na base e secret |
| Falha ao habilitar TRANDATA | CDC não habilitado ou privilégio insuficiente | Habilitar CDC com DBA ou conceder `sysadmin` temporariamente |
| Job CDC não executa | SQL Server Agent parado | Iniciar SQL Server Agent e validar seus jobs |
| `unable to access secrets using resource principal` | Dynamic Group ou policy ausente | Revisar `read secret-bundles` |
| Connection retorna `Invalid walletSecretId` | Wallet secret informado para SQL Server | Remover wallet secret; use somente password secret, salvo requisito TLS específico |

## Referências

- [Connect to Microsoft SQL Server — OCI GoldenGate](https://docs.oracle.com/en/cloud/paas/goldengate-service/qodtl/)
- [Prepare Database Users and Privileges for SQL Server](https://docs.oracle.com/en/database/goldengate/core/26/coredoc/prepare-database-users-and-privileges-oracle-goldengate-processes-sql-server.html)
- [Prepare Database Connection for SQL Server](https://docs.oracle.com/en/database/goldengate/core/26/coredoc/prepare-database-connection-system-and-parameter-settings-sql-server.html)
- [OCI GoldenGate Policies](https://docs.oracle.com/en/cloud/paas/goldengate-service/ocigg/oracle-cloud-infrastructure-goldengate-policies.html)
