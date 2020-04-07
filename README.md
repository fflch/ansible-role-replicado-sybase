# Ansible Role: SAP ASE

Esta *role* encapsula parte da toda lógica da criação de um servidor
*SAP Sybase ASE Express Edition for Linux* e foi baseada em [ansible-sap-iq](https://github.com/andrewrothstein/ansible-sap-iq).

## Recursos
 
 - Cria um usuário e grupo sybase
 - Configura ulimit para unlimited
 - Instala dependências necessárias: libaio1, unzip
 - Copia e descompacta o pacote sybase .tgz no servidor
 - Provê um arquivo *response* com uma parametrização básica para instalação do SAP Sybase ASE
 - Permite especificar o caminho do log de serviço do servidor
 - Configura o comando service sybase start (ou stop)

## Pré-configurações

1. Colocar o tar no servidor
2. separar duas partições: /replicado e /replicado/log?

## Algumas configurações pós-instalação com o isql

Conectar no servidor estando no mesmo, o -w1000 torna os outputs 
mas *bonitos*:

    isql -Usa -PSUA_SENHA -SSERVERNAME -w1000

Conectar no servidor remotamente:

    isql -Usa -PSUA_SENHA -S192.168.100.14 -w1000

Ver e alterar o número de locks:

    1> sp_configure 'number of locks'
    2> go
    1> sp_configure 'number of locks', 100000 
    2> go
    
Ver o alterar o número de conexões que o servidor recebe:

    1> sp_configure 'number of user connections'
    2> go
    1> sp_configure 'number of user connections', 150
    2> go

Criar usuário dbmaint. Lista todos usuários:

    1> use master
    2> go
    1> sp_addlogin "dbmaint", "SuaSuperSenha123"
    2> go
    1> select name from syslogins
    2> go

Criar um banco chamado fflch_dbc. Lista databases:

    1> use master
    2> go
    disk init 
    name = "fflch_data", 
    physname = "/replicado/bin/data/fflch_data.dat", 
    size = "40G"

    disk init
    name = "fflch_log", 
    physname = "/replicado/bin/data/fflch_log.dat", 
    size = "40G"

    CREATE DATABASE fflch ON fflch_data='40G' LOG ON fflch_log='40G'
    GO

    1> sp_helpdb	
    2> go
    1> sp_helpdb fflch	
    2> go

Ativar a opção de truncate log on checkpoint para o banco de dados fflch_dbc:

    1> use master
    2> go
    1> sp_dboption fflch, "trunc log on chkpt", true
    2> go
    
Usuário dbmaint com privilégio de criar tabelas no Banco de Dados (alias de dbo):

    1> use fflch
    2> go
    1> sp_addalias dbmaint,dbo
    2> go
    1> sp_helpuser
    2> go

## Dicas

Mostrar tabelas do banco de dados fflch:

    1> use fflch
    2> go
    1> exec sp_tables '%', '%', 'fflch',"'TABLE'"
    2> go

Outra forma de mostrar tabelas do banco de dados fflch:

    1> use fflch
    2> go 
    1> select name from sysobjects where type = 'U' or type = 'P'
    2> go 

Carregar um arquivo sql:

    isql -Usa -PSuaSenha -SSEU_SERVER -iSeuArquivoComSql.sql

Mostrar qual banco de dados que estou connectado no momento:

    1> select db_name()
    2> go
       
Dicas de IDEs para administrar graficamente o banco: 

- https://dbeaver.io/download/


### Criando um usuário com permissão de somente leitura

Criar login e senha:

    1> use master
    2> go
    1> sp_addlogin "read_only_user", "SUA_SENHA"
    2> go

Adicionar usuário acima ao banco fflch:

    1> use fflch
    2> go
    1> sp_adduser 'read_only_user', 'read_only_user'
    2> go

Gerar SQLs que darão permissão de select em todas tabelas do banco fflch:

    1> use fflch
    2> go
    1> select 'grant select on ' + name + ' to read_only_user' from sysobjects where type = 'U' or type = 'P'
    2> go

Testar conexã do novo usuário:

    isql -Uread_only_user -PSUA_SENHA -SSYBASE -w1000
