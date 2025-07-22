Setup de replicação no Postgres
==============================

- Este manual exemplifica como fazer a configuração de replicação no postgres, de maneira manual.
- São considerados ao menos 2 servidores, mas, podem ser adicionados mais.

- Nota: a leitura e escrita é feita apenas no principal.

Conceitos fundamentais de replicação
-----------------------------------

- **Write-Ahead Log (WAL)**: Garante a integridade dos dados e durabilidade das transações. As transações são registradas em um log de transações, o WAL, antes que sejam gravadas em disco. Na replicação, o WAL é o fluxo de dados que é enviado do servidor primário para os servidores secundários.

- **Log Sequence Number (LSN)**: é um ponteiro para uma localização específica dentro do fluxo do WAL.

- **Slots de replicação**: recurso que garante que o servidor primário não remova segmentos do WAL que ainda são necessários por uma ou mais réplicas, mesmo que essas réplicas estejam temporariamente desconectadas.

Tipos de replicação
-------------------

- No Postgres temos diversas opções de replicação.

Replicação Física (streaming replication)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Esse tipo de replicação trabalha no nível do bloco de dados, copiando o fluxo de WAL do primário para as réplicas.
- As réplicas são cópias exatas do primário e são ideais para alta disponibilidade e balanceamento de carga de leitura. É essa replicação que utilizaremos no exemplo.
- Pode ser **assíncrona** ou **síncrona**:

  - **Replicação Assíncrona**: O servidor primário não espera que a réplica confirme o recebimento e a aplicação dos dados do WAL antes de confirmar uma transação.

  - **Replicação Síncrona**: O servidor primário espera que a réplica confirme o recebimento e a gravação dos dados do WAL antes de confirmar uma transação.

Replicação Lógica
~~~~~~~~~~~~~~~~~

- Trabalha em um nível superior, replicando alterações de dados com base em sua identidade de replicação (geralmente uma chave primária). Ela usa um modelo de publicação/assinatura, onde um servidor primário (publicador) publica alterações para um ou mais servidores secundários (assinantes). 
- Diferente da replicação física, a replicação lógica permite:

  - Replicar subconjuntos de dados (tabelas específicas, bancos de dados específicos).
  - Replicar entre diferentes versões do PostgreSQL.
  - Replicar entre diferentes arquiteturas de hardware.
  - Maior flexibilidade para transformações de dados durante a replicação.

- Usaremos a replicação física por ser mais simples de configurar.

Configurações do Servidor Primário
---------------------------------

- Para configurar o servidor primário, que será o ponto de escrita e origem dos dados replicados, seguiremos os passos abaixo utilizando Docker Compose para facilitar a criação do ambiente.

1. Estrutura de Arquivos
~~~~~~~~~~~~~~~~~~~~~~~~

- Certifique-se de ter a seguinte estrutura de diretório e arquivos:

  .. code-block:: bash

     .env
     docker-compose.yml
     init-replication-user.sh
     pg_hba.conf
     postgresql.conf
     volumes/
     └── postgres/
         └── data/ (será criado pelo Docker)

2. Arquivo docker-compose.yml
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Crie o arquivo ``docker-compose.yml``. Ele define o serviço do Postgres primário, mapeia as portas, volumes e variáveis de ambiente necessárias.

  .. code-block:: yaml

     networks:
       postgres:
         external: true
         name: postgres
       reverse-proxy:
         external: true
         name: reverse-proxy
         
     services:
       postgres:
         ports:
           - 5432:5432
         container_name: postgres-postgres-1
         image: postgres:16-alpine
         restart: always
         volumes:
           - ./volumes/postgres/data:/var/lib/postgresql/data
           - ./.docker/postgres/init-user-db.sh:/docker-entrypoint-initdb.d/init-user-db.sh
           - ./postgresql.conf:/var/lib/postgresql/postgresql.conf:rw
           - ./pg_hba.conf:/var/lib/postgresql/data/pg_hba.conf:rw
         networks:
           - postgres
         environment:
           - POSTGRES_PASSWORD
           - POSTGRES_DB
           - POSTGRES_USER
           - TZ
           - POSTGRES_REPLICATION_USER
           - POSTGRES_REPLICATION_PASSWORD

3. Script init-replication-user.sh
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Crie o script ``init-replication-user.sh`` responsável por criar o usuário de replicação assim que o container for inicializado:

  .. code-block:: bash

     #!/bin/bash
     set -e

     psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
         CREATE USER $POSTGRES_REPLICATION_USER WITH REPLICATION ENCRYPTED PASSWORD '$POSTGRES_REPLICATION_PASSWORD';
         SELECT pg_create_physical_replication_slot('replication_slot');
     EOSQL

4. Arquivo .env
~~~~~~~~~~~~~~~

- Crie o arquivo ``.env`` e substitua as variáveis de acordo com seu ambiente:

  .. code-block:: bash

     POSTGRES_USER=postgres #Nome do usuário do postgres
     POSTGRES_PASSWORD=example #Senha do usuário - Alterar
     POSTGRES_DB=postgres #Nome do banco de dados

     POSTGRES_REPLICATION_USER=usuario_replicacao #Nome do usuário de replicação
     POSTGRES_REPLICATION_PASSWORD=senha_replicacao #Senha do usuário de replicação
     TZ=America/Sao_Paulo

5. Arquivo postgresql.conf
~~~~~~~~~~~~~~~~~~~~~~~~~~

- Crie o arquivo ``postgresql.conf`` com o seguinte conteúdo:

  .. code-block:: ini

     # Configurações básicas do Postgres
     listen_addresses = '*'
     max_connections = 100
     shared_buffers = 128MB
     dynamic_shared_memory_type = posix

     # Configurações de Write-Ahead Log (WAL) e Réplica
     wal_level = replica
     max_wal_senders = 10
     wal_keep_size = 1GB
     max_replication_slots = 10

     # Outras configurações importantes
     checkpoint_timeout = 5min
     max_wal_size = 1GB
     min_wal_size = 80MB

6. Arquivo pg_hba.conf
~~~~~~~~~~~~~~~~~~~~~~

- No arquivo ``pg_hba.conf`` especifique quais hosts podem se conectar:

  .. code-block::

     # TYPE  DATABASE        USER                    ADDRESS                        METHOD
     local   all             all                                                    trust
     host    all             all                     127.0.0.1/32                   trust
     host    all             all                     172.0.0.0/8                    trust
     host    all             usuario_replicacao      IP_Servidor_Secundário/32      md5

- Caso seja necessário replicar para outro servidor, acrescente outra linha modificando/acrescentando o endereço da outra réplica.

.. note::
   Está sendo feita a replicação de todos os bancos. Em caso de replica somente de um banco específico, mude a variável ``all`` para o nome do seu banco.

7. Iniciar o container
~~~~~~~~~~~~~~~~~~~~~~

- Inicie o container:

  .. code-block:: bash

     docker compose up -d

- Verifique se o container foi iniciado corretamente:

  .. code-block:: bash

     docker compose exec -ti postgres psql -U postgres -c '\du;'

Servidor secundário
-------------------

1. Arquivo .env
~~~~~~~~~~~~~~~

- Crie o arquivo ``.env``:

  .. code-block:: bash

     POSTGRES_PASSWORD= #Senha do banco atual
     POSTGRES_PRIMARY_HOST= #Endereço IP ou DNS do servidor primário
     POSTGRES_PRIMARY_PORT=5432
     POSTGRES_REPLICATION_USER= #Nome do usuário de replicação
     POSTGRES_REPLICATION_PASSWORD= # Senha do usuário de replicação

2. Cópia inicial dos dados (pg_basebackup)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Execute os comandos:

  .. code-block:: bash

     rm -rf ./volumes/postgres/data/*
     docker run --rm -v ./volumes/postgres/data:/var/lib/postgresql/data postgres:16-alpine \
       bash -c "PGPASSWORD=$POSTGRES_REPLICATION_PASSWORD pg_basebackup -h $POSTGRES_PRIMARY_HOST -p $POSTGRES_PRIMARY_PORT -U $POSTGRES_REPLICATION_USER -D /var/lib/postgresql/data -P -R -v --wal-method=stream"

3. Arquivo compose.yml
~~~~~~~~~~~~~~~~~~~~~~

- Crie o arquivo ``compose.yml``:

  .. code-block:: yaml

     networks:
       postgres:
         external: true
         name: postgres

     services:
       postgres:
         container_name: postgres
         image: postgres:16-alpine
         restart: always
         volumes:
           - ./volumes/postgres/data:/var/lib/postgresql/data
           - ./volumes/postgresql.conf:/etc/postgresql/postgresql.conf
           - ./pg_hba.conf:/var/lib/postgresql/data/pg_hba.conf 
         networks:
           - postgres
         environment:
           - POSTGRES_PASSWORD
           - POSTGRES_REPLICATION_USER
           - POSTGRES_REPLICATION_PASSWORD
           - POSTGRES_PRIMARY_HOST
           - POSTGRES_PRIMARY_PORT
         ports:
           - "5432:5432"

- Suba o container:

  .. code-block:: bash

     docker compose up -d

Verificar a replicação
----------------------

- No **servidor primário**:

  .. code-block:: bash

     docker exec -ti postgres-postgres-1 psql -U postgres -c "SELECT * FROM pg_stat_replication;"

- Na **réplica**:

  .. code-block:: bash

     docker exec -ti postgres-postgres-1 psql -U postgres -c "SELECT pg_is_in_recovery();"

- Ver progresso da replicação:

  .. code-block:: bash

     docker exec -ti postgres psql -U postgres -c "SELECT now() AS current_time, pg_last_wal_receive_lsn() AS receive_lsn, pg_last_wal_replay_lsn() AS replay_lsn pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS replay_delay;"

Replica para Primária
---------------------

- Promover a réplica:

  .. code-block:: bash

     docker exec -ti postgres-postgres-1 psql -U postgres -c "SELECT pg_promote();"
     # ou
     docker exec -ti postgres-postgres-1 pg_ctl promote -D /var/lib/postgresql/data

.. note::
   Se o antigo primário voltar a funcionar, devemos realizar o processo utilizando o ``pg_basebackup`` à partir do antigo primário e configurar a replicação para o mesmo.