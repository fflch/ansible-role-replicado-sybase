# Ansible Role: SAP ASE

Esta *role* encapsula parte da toda lógica da criação de um servidor
*SAP Sybase ASE Express Edition for Linux* e foi baseada em [ansible-sap-iq](https://github.com/andrewrothstein/ansible-sap-iq).

Foi testado em:

 - Debian 9
 - Debian 10

## Recursos
 
 - Seleciona a versão express
 - Cria um usuário e grupo sybase, deixando como sudo
 - Configura ulimit para unlimited
 - Instala dependências necessárias, como libaio1, unzip
 - Parte do pressuposto que você baixou o .tgz no servidor
 - Provê um arquivo *response* com uma parametrização básica para instalação do SAP Sybase ASE
 - Permite especificar o caminho do log de serviço do servidor
 - Configura o service *sybase* com start e stop
 - Service *sybase* inicializa automaticamente no boot
 - SAP ase server sobre na porta 5000 e backup server na porta 5001
 - As variáveis de ambiente para usar o isql estão globais para todos usuários do sistema

# Instalação

O *response file* usado neste role está disponível em *templates/response.j2*
e por padrão habilitar o servidor SAP ASE e o servidor de Backup. Para
habilitar outros serviços, esse arquivo pode ser alterado.

Nesta instalação, vamos isolar a partição do sap ase
deixando um diretório para os dados e outra para log:
 */replicado* e */replicado/log*, assim, especifique as
variáveis do ansible correspondente:

    sap_ase_home: '/replicado'
    sap_ase_log: '/replicado/log'

Baixar o arquivo *.tgz* oficial do sybase e colocá-lo no seu servidor.
Especifique a variável do ansible correspondente:

    sap_ase_tar: '/root/ASE_Suite.linuxamd64.tgz'

Segue o playbook completo com as demais variáveis que devem ser configuradas:

    - name: deploy
      become: yes
      hosts: sybase

      tasks:

        - name: fflch.sap-ase
          include_role:
            name: fflch.sap-ase
          vars:
            sap_ase_home: '/replicado'
            sap_ase_log: '/replicado/log'
            sap_ase_tar: '/root/ASE_Suite.linuxamd64.tgz'
     
            sap_ase_host: 'replicado.fflch.usp.br'
            sap_ase_password: "SenhaDoServidor123"

Seu servidor foi instalado em /replicado/sap.

# Algumas configurações pós-instalação

Acessar servidor sybase estando conectado via ssh,
com o usuário *sa*.
O *-w1000* torna os outputs mas *bonitos*:

    isql -Usa -PSUA_SENHA -SSYBASE -w1000

Acessar servidor sybase remotamente, com usuário *sa*, 
dado que você tem o isql localmente:

    isql -Usa -PSUA_SENHA -S192.168.8.56 -w1000

(recomendação replicado) Ver e alterar o número de locks:

    1> sp_configure 'number of locks'
    2> go
    1> sp_configure 'number of locks', 100000 
    2> go
    
(recomendação replicado) Ver o alterar o número de conexões:

    1> sp_configure 'number of user connections'
    2> go
    1> sp_configure 'number of user connections', 150
    2> go

(recomendação replicado) Criar usuário dbmaint.

    1> use master
    2> go
    1> sp_addlogin "dbmaint", "SuaSuperSenha123"
    2> go

Lista todos usuários:

    1> use master
    2> go
    1> select name from syslogins
    2> go

A versão express nos permite usar 100G apenas. Se você usou
o response file desta *role*, foi configurado os seguintes discos
no sap ase:

    master: 2GB
    sysprocsdev: 4GB
    systemdbdev: 2GB
    tempdbdev: 2GB

Vamos deixa 10GB disponível para usos futuros.
Nos resta 80GB que vamos usar para *data* e *log*.

(recomendação replicado) Criando disco de 50G para dados, que chamaremos de *fflch_data*:

    1> use master
    2> go

    1> disk init 
    name = "fflch_data", 
    physname = "/replicado/sap/data/fflch_data.dat", 
    size = "50G"
    5> go

(recomendação replicado) Vamos usar 30G para log, que chamaremos de *fflch_log*:

    1> use master
    2> go
    
    1> disk init
    name = "fflch_log", 
    physname = "/replicado/log/fflch_log.dat", 
    size = "30G"
    5> go 
    
Criando banco de dados nos discos criados e alocando todo espaço disponível:

    CREATE DATABASE fflch ON fflch_data='50G' LOG ON fflch_log='30G'
    GO

Checar os *devices* criados:

    1> sp_helpdevice
    2> go

Listar bancos de dados:

    1> sp_helpdb	
    2> go
    1> sp_helpdb fflch	
    2> go

(recomendação replicado) Ativar a opção de truncate log on 
checkpoint para o banco de dados fflch:

    1> use master
    2> go
    1> sp_dboption fflch, "trunc log on chkpt", true
    2> go

(recomendação replicado) Deixar usuário dbmaint com privilégio de 
criar tabelas no banco de dados *fflch* (alias de dbo):

    1> use fflch
    2> go
    1> sp_addalias dbmaint,dbo
    2> go
    1> sp_helpuser
    2> go

## Dicas de uso geral

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

Mostra em qual banco de dados que estou connectado no momento:

    1> select db_name()
    2> go

Carregar um arquivo sql:

    isql -Usa -PSuaSenha -SYBASE -iSeuArquivoComSql.sql
       
Dicas de IDE para fazer queries graficamente no banco: 

 - https://dbeaver.io

Criando um usuário *fulano* com senha:

    1> use master
    2> go
    1> sp_addlogin "fulano", "SUA_SENHA"
    2> go

Adicionar usuário acima ao banco fflch:

    1> use fflch
    2> go
    1> sp_adduser 'fulano', 'fulano'
    2> go

Testar conexão do novo usuário:

    isql -Ufulano -PSUA_SENHA -SSYBASE -w1000

Gerar SQLs, que podem ser copiadas e aplicadas, que darão 
permissão de SELECT em todas tabelas do banco fflch:

    1> use fflch
    2> go
    1> select 'grant select on ' + name + ' to fulano' from sysobjects where type = 'U' or type = 'P'
    2> go

Gerar um dump do banco de dados:

    isql -Usa -PSuaSenha -SYBASE -w1000
    1> use master
    2> go
    1> dump database fflch to "/replicado/fflch.dump"
    2> go

(não testado) Gerar um dump do banco de dados desligando o log banco antes:

    isql -Usa -PSuaSenha -SYBASE -w1000
    1> use master
    2> go

    1> alter database fflch log off fflch_log
    2> go

    1> dump database fflch to "/replicado/fflch.dump"
    2> go

    1> online database fflch
    2> go

Carregar banco de dados a partir de um dump:

    isql -Usa -PSuaSenha -SYBASE -w1000
    1> use master
    2> go
    1> load database fflch from "/replicado/fflch.dump"
    2> go
