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

     POSTGRES_PASSWORD=senha_postgres    # Senha do usuário postgres
     POSTGRES_PRIMARY_HOST=192.168.1.10  # Endereço IP ou DNS do servidor primário
     POSTGRES_PRIMARY_PORT=5432
     POSTGRES_REPLICATION_USER=usuario_replicacao  # Nome do usuário de replicação
     POSTGRES_REPLICATION_PASSWORD=senha_replicacao  # Senha do usuário de replicação
     TZ=America/Sao_Paulo

2. Criar estrutura de diretórios
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Crie a estrutura necessária:

  .. code-block:: bash

     mkdir -p volumes/postgres/data
     chmod 700 volumes/postgres

3. Cópia inicial dos dados (pg_basebackup)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Execute o ``pg_basebackup`` para copiar os dados do primário:

  .. code-block:: bash

     rm -rf ./volumes/postgres/data/*
     docker run --rm -v ./volumes/postgres/data:/var/lib/postgresql/data postgres:16-alpine \
       bash -c "PGPASSWORD=$POSTGRES_REPLICATION_PASSWORD pg_basebackup -h $POSTGRES_PRIMARY_HOST -p $POSTGRES_PRIMARY_PORT -U $POSTGRES_REPLICATION_USER -D /var/lib/postgresql/data -P -R -v --wal-method=stream"

- Parâmetros do ``pg_basebackup``:

  - ``-h``: Host do servidor primário
  - ``-p``: Porta do servidor primário
  - ``-U``: Usuário de replicação
  - ``-D``: Diretório de destino
  - ``-P``: Mostra progresso
  - ``-R``: Cria automaticamente ``postgresql.auto.conf`` com configurações de replicação
  - ``-v``: Modo verboso
  - ``--wal-method=stream``: Transmite WAL durante o backup

4. Verificar configuração criada automaticamente
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Após o ``pg_basebackup``, verificar o arquivo ``postgresql.auto.conf``:

  .. code-block:: bash

     cat ./volumes/postgres/data/postgresql.auto.conf

- Deve conter algo como:

  .. code-block:: ini

     primary_conninfo = 'user=usuario_replicacao password=senha_replicacao host=192.168.1.10 port=5432 sslmode=prefer'
     primary_slot_name = 'replication_slot'

5. Arquivo compose.yml
~~~~~~~~~~~~~~~~~~~~~~

- Crie o arquivo ``compose.yml``:

  .. code-block:: yaml

     networks:
       postgres:
         external: true
         name: postgres

     services:
       postgres-replica:
         container_name: postgres-replica
         image: postgres:16-alpine
         restart: always
         volumes:
           - ./volumes/postgres/data:/var/lib/postgresql/data
         networks:
           - postgres
         environment:
           - POSTGRES_PASSWORD
           - TZ
         ports:
           - "5432:5432"
         healthcheck:
           test: ["CMD-SHELL", "pg_isready -U postgres"]
           interval: 10s
           timeout: 5s
           retries: 5
           start_period: 30s

.. note::
   - O ``pg_basebackup`` com flag ``-R`` já cria o arquivo ``postgresql.auto.conf`` com as configurações de replicação
   - Não é necessário montar volumes adicionais de configuração
   - O ``pg_hba.conf`` é copiado do primário durante o ``pg_basebackup``

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

     docker exec -ti postgres-replica psql -U postgres -c "SELECT pg_is_in_recovery();"

- Ver progresso da replicação:

  .. code-block:: bash

     docker exec -ti postgres-replica psql -U postgres -c "SELECT now() AS current_time, pg_last_wal_receive_lsn() AS receive_lsn, pg_last_wal_replay_lsn() AS replay_lsn, pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS replay_delay;"

Monitoramento Avançado da Replicação
-------------------------------------

Verificar lag de replicação
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Monitorar o atraso da replicação em bytes e tempo:

  .. code-block:: bash

     docker exec -ti postgres-postgres-1 psql -U postgres -c "
     SELECT 
       client_addr,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes,
       write_lag,
       flush_lag,
       replay_lag
     FROM pg_stat_replication;"

- Verificar tamanho dos WAL files acumulados:

  .. code-block:: bash

     docker exec -ti postgres-postgres-1 psql -U postgres -c "
     SELECT 
       slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
     FROM pg_replication_slots;"

.. warning::
   Se o ``retained_wal`` crescer muito, pode causar falta de espaço em disco. Configure alertas para valores acima de 10GB.

Monitorar espaço em disco
~~~~~~~~~~~~~~~~~~~~~~~~~

- Verificar uso de disco no diretório WAL:

  .. code-block:: bash

     docker exec -ti postgres-postgres-1 sh -c "du -sh /var/lib/postgresql/data/pg_wal"

- Verificar inodes disponíveis:

  .. code-block:: bash

     docker exec -ti postgres-postgres-1 df -i /var/lib/postgresql/data

Script de monitoramento contínuo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Criar um script para monitorar a saúde da replicação:

  .. code-block:: bash

     #!/bin/bash
     # monitor-replication.sh
     
     THRESHOLD_BYTES=104857600  # 100MB
     THRESHOLD_SECONDS=60       # 60 segundos
     
     LAG=$(docker exec postgres-postgres-1 psql -U postgres -t -c "
       SELECT COALESCE(pg_wal_lsn_diff(sent_lsn, replay_lsn), 0)
       FROM pg_stat_replication LIMIT 1;"
     )
     
     if [ "$LAG" -gt "$THRESHOLD_BYTES" ]; then
       echo "ALERTA: Replicação atrasada em $LAG bytes"
       # Enviar notificação (e-mail, webhook, etc)
     fi

Replica para Primária
---------------------

- Promover a réplica:

  .. code-block:: bash

     docker exec -ti postgres-postgres-1 psql -U postgres -c "SELECT pg_promote();"
     # ou
     docker exec -ti postgres-postgres-1 pg_ctl promote -D /var/lib/postgresql/data

.. note::
   Se o antigo primário voltar a funcionar, devemos realizar o processo utilizando o ``pg_basebackup`` à partir do antigo primário e configurar a replicação para o mesmo.

Replicação Síncrona (Zero Data Loss)
-------------------------------------

- A replicação síncrona garante que as transações só são confirmadas após serem gravadas na réplica, eliminando perda de dados em caso de falha.

Configuração no Servidor Primário
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Edite o arquivo ``postgresql.conf`` e adicione:

  .. code-block:: ini

     # Replicação Síncrona
     synchronous_commit = on
     synchronous_standby_names = 'replication_slot'  # Nome do slot ou 'ANY 1 (*)' para qualquer réplica

- Opções para ``synchronous_standby_names``:

  - ``'replication_slot'``: Aguarda réplica específica
  - ``'ANY 1 (*)'``: Aguarda qualquer 1 réplica disponível
  - ``'FIRST 2 (replica1, replica2, replica3)'``: Aguarda as 2 primeiras réplicas da lista

.. warning::
   A replicação síncrona aumenta a latência das escritas. Use apenas quando a perda de dados for inaceitável.

Verificar modo síncrono
~~~~~~~~~~~~~~~~~~~~~~~

- No servidor primário:

  .. code-block:: bash

     docker exec -ti postgres-postgres-1 psql -U postgres -c "
     SELECT 
       application_name,
       client_addr,
       state,
       sync_state,
       sync_priority
     FROM pg_stat_replication;"

- O campo ``sync_state`` deve mostrar ``sync`` para réplica síncrona ativa.

Segurança da Replicação
-----------------------

Configuração de Autenticação Segura
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Substituir ``trust`` por ``scram-sha-256`` no arquivo ``pg_hba.conf``:

  .. code-block::

     # TYPE  DATABASE        USER                    ADDRESS                        METHOD
     local   all             all                                                    scram-sha-256
     host    all             all                     127.0.0.1/32                   scram-sha-256
     host    all             all                     172.0.0.0/8                    scram-sha-256
     host    replication     usuario_replicacao      IP_Servidor_Secundário/32      scram-sha-256

- Criar senha com hash SCRAM-SHA-256:

  .. code-block:: bash

     docker exec -ti postgres-postgres-1 psql -U postgres -c "
     ALTER USER usuario_replicacao PASSWORD 'senha_replicacao';
     ALTER SYSTEM SET password_encryption = 'scram-sha-256';"

Configuração SSL/TLS
~~~~~~~~~~~~~~~~~~~~

- Gerar certificados SSL:

  .. code-block:: bash

     # No servidor primário
     openssl req -new -x509 -days 365 -nodes -text \
       -out server.crt -keyout server.key -subj "/CN=postgres-primary"
     chmod 600 server.key
     chown 70:70 server.key server.crt  # UID do postgres no container

- Adicionar ao ``postgresql.conf``:

  .. code-block:: ini

     ssl = on
     ssl_cert_file = '/var/lib/postgresql/server.crt'
     ssl_key_file = '/var/lib/postgresql/server.key'

- Atualizar ``pg_hba.conf`` para exigir SSL:

  .. code-block::

     hostssl replication usuario_replicacao IP_Servidor_Secundário/32 scram-sha-256

- Configurar réplica para usar SSL no arquivo ``postgresql.auto.conf``:

  .. code-block:: ini

     primary_conninfo = 'host=IP_Primario port=5432 user=usuario_replicacao password=senha_replicacao sslmode=require'

Firewall e Segurança de Rede
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Configurar firewall no servidor primário:

  .. code-block:: bash

     # Permitir apenas IP da réplica
     iptables -A INPUT -p tcp -s IP_Servidor_Secundário --dport 5432 -j ACCEPT
     iptables -A INPUT -p tcp --dport 5432 -j DROP

- Ou usando UFW:

  .. code-block:: bash

     ufw allow from IP_Servidor_Secundário to any port 5432
     ufw deny 5432

Solução de Problemas
--------------------

Réplica não está sincronizando
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Verificar conectividade de rede:

   .. code-block:: bash

      # Na réplica, testar conexão ao primário
      docker exec -ti postgres psql -h IP_Primario -U usuario_replicacao -d postgres -c "SELECT 1;"

2. Verificar logs da réplica:

   .. code-block:: bash

      docker logs postgres --tail 100

3. Verificar se o slot de replicação existe no primário:

   .. code-block:: bash

      docker exec -ti postgres-postgres-1 psql -U postgres -c "SELECT * FROM pg_replication_slots;"

4. Se o slot estiver inativo, recriar a réplica com ``pg_basebackup``.

Slot de replicação preenchendo disco
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Se a réplica ficar offline por muito tempo, o slot acumula WAL files:

  .. code-block:: bash

     # Verificar slots inativos
     docker exec -ti postgres-postgres-1 psql -U postgres -c "
     SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
     FROM pg_replication_slots;"

- Remover slot se a réplica não será recuperada:

  .. code-block:: bash

     docker exec -ti postgres-postgres-1 psql -U postgres -c "
     SELECT pg_drop_replication_slot('replication_slot');"

.. warning::
   Remover o slot significa que a réplica precisará ser recriada do zero com ``pg_basebackup``.

Conflitos após promoção de réplica
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Após promover uma réplica, o antigo primário não pode voltar como réplica automaticamente.

- **Processo de re-sincronização**:

  1. Parar o antigo primário:

     .. code-block:: bash

        docker compose down

  2. Limpar dados antigos:

     .. code-block:: bash

        rm -rf ./volumes/postgres/data/*

  3. Fazer ``pg_basebackup`` do novo primário:

     .. code-block:: bash

        docker run --rm -v ./volumes/postgres/data:/var/lib/postgresql/data postgres:16-alpine \
          bash -c "PGPASSWORD=$POSTGRES_REPLICATION_PASSWORD pg_basebackup -h IP_NOVO_PRIMARIO -p 5432 -U $POSTGRES_REPLICATION_USER -D /var/lib/postgresql/data -P -R -v --wal-method=stream"

  4. Reiniciar como réplica:

     .. code-block:: bash

        docker compose up -d

Erro: "requested WAL segment has already been removed"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Significa que a réplica está muito atrasada e os WAL files necessários já foram removidos.

- **Solução**: Recriar a réplica com ``pg_basebackup``.

- **Prevenção**: Aumentar ``wal_keep_size`` no ``postgresql.conf`` ou usar ``wal_keep_segments`` em versões antigas:

  .. code-block:: ini

     wal_keep_size = 5GB  # Aumentar conforme necessário

Réplica em recovery contínuo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Se a réplica não sai do modo recovery:

  .. code-block:: bash

     # Verificar se há erros no log
     docker logs postgres --tail 200 | grep -i error

     # Verificar LSN atual
     docker exec -ti postgres psql -U postgres -c "
     SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();"

- Se o LSN não avança, verificar:

  - Conectividade com o primário
  - Credenciais do usuário de replicação
  - Configuração do arquivo ``postgresql.auto.conf``

Backup e Recuperação
--------------------

Arquivamento de WAL (WAL Archiving)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Configurar arquivamento contínuo de WAL para recuperação point-in-time (PITR).

- No ``postgresql.conf`` do servidor primário:

  .. code-block:: ini

     archive_mode = on
     archive_command = 'test ! -f /var/lib/postgresql/wal_archive/%f && cp %p /var/lib/postgresql/wal_archive/%f'
     archive_timeout = 300  # Arquivar a cada 5 minutos mesmo sem atividade

- Adicionar volume no ``docker-compose.yml``:

  .. code-block:: yaml

     volumes:
       - ./volumes/postgres/data:/var/lib/postgresql/data
       - ./volumes/postgres/wal_archive:/var/lib/postgresql/wal_archive

- Criar diretório:

  .. code-block:: bash

     mkdir -p ./volumes/postgres/wal_archive
     chmod 700 ./volumes/postgres/wal_archive

Backup com pg_basebackup
~~~~~~~~~~~~~~~~~~~~~~~~

- Fazer backup completo do cluster:

  .. code-block:: bash

     # Backup do primário
     docker run --rm -v ./backup:/backup postgres:16-alpine \
       bash -c "PGPASSWORD=$POSTGRES_PASSWORD pg_basebackup -h IP_Primario -U postgres -D /backup/$(date +%Y%m%d_%H%M%S) -Ft -z -P"

- Backup incluirá:

  - Todos os bancos de dados
  - Configurações
  - Tablespaces
  - WAL necessário para consistência

Point-in-Time Recovery (PITR)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Recuperar banco para um momento específico no tempo.

1. Restaurar backup base:

   .. code-block:: bash

      rm -rf ./volumes/postgres/data/*
      tar -xzf backup/20250122_120000/base.tar.gz -C ./volumes/postgres/data/

2. Criar arquivo ``recovery.signal``:

   .. code-block:: bash

      touch ./volumes/postgres/data/recovery.signal

3. Configurar ``postgresql.conf`` ou ``postgresql.auto.conf``:

   .. code-block:: ini

      restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'
      recovery_target_time = '2025-01-22 12:30:00'
      recovery_target_action = 'promote'

4. Iniciar o container:

   .. code-block:: bash

      docker compose up -d

5. Verificar logs para confirmar recuperação:

   .. code-block:: bash

      docker logs postgres | grep -i recovery

Backup com ferramentas externas
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Para backup de réplicas sem impacto no primário:

  .. code-block:: bash

     # Usar réplica como fonte de backup
     docker exec postgres-replica pg_dump -U postgres nome_banco > backup_$(date +%Y%m%d).sql

- Integração com Duplicati (mencionado na documentação):

  - Configurar backup dos volumes de dados
  - Incluir ``pg_wal`` e ``wal_archive`` se configurado
  - Fazer dump SQL regularmente para backups lógicos

Balanceamento de Carga e Otimização
------------------------------------

PgBouncer para Connection Pooling
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Instalar PgBouncer na frente do Postgres:

  .. code-block:: yaml

     # Adicionar ao docker-compose.yml
     services:
       pgbouncer:
         image: edoburu/pgbouncer:latest
         container_name: pgbouncer
         environment:
           - DATABASE_URL=postgres://postgres:senha@postgres:5432/postgres
           - POOL_MODE=transaction
           - MAX_CLIENT_CONN=1000
           - DEFAULT_POOL_SIZE=25
         ports:
           - "6432:5432"
         networks:
           - postgres

- Conectar Nextcloud ao PgBouncer (porta 6432) em vez do Postgres direto.

- Benefícios:

  - Reduz overhead de conexões
  - Protege banco contra excesso de conexões
  - Melhora performance do Nextcloud

Balanceamento de leitura entre réplicas
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Configurar HAProxy para distribuir leituras:

  .. code-block::

     # haproxy.cfg
     listen postgres_read
         bind *:5433
         mode tcp
         balance leastconn
         option tcp-check
         server replica1 IP_Replica1:5432 check
         server replica2 IP_Replica2:5432 check backup

- Nextcloud pode usar:

  - Porta 5432 (via PgBouncer) para escritas no primário
  - Porta 5433 (via HAProxy) para leituras distribuídas

.. note::
   O Nextcloud por padrão não suporta separação read/write nativamente. Seria necessário configuração customizada ou usar apenas réplicas para relatórios/backups.

Otimização de parâmetros para Nextcloud
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Ajustes recomendados no ``postgresql.conf`` para workload do Nextcloud:

  .. code-block:: ini

     # Memória
     shared_buffers = 256MB          # 25% da RAM disponível
     effective_cache_size = 1GB      # 50-75% da RAM
     work_mem = 16MB                 # Para sorts e joins
     maintenance_work_mem = 128MB    # Para VACUUM, CREATE INDEX

     # Checkpoints
     checkpoint_completion_target = 0.9
     wal_buffers = 16MB

     # Planner
     random_page_cost = 1.1          # Para SSDs (padrão 4.0 para HDDs)
     effective_io_concurrency = 200  # Para SSDs

     # Autovacuum (importante para Nextcloud)
     autovacuum = on
     autovacuum_max_workers = 3
     autovacuum_naptime = 10s

- Reiniciar para aplicar:

  .. code-block:: bash

     docker compose restart postgres

Manutenção preventiva
~~~~~~~~~~~~~~~~~~~~~

- Executar VACUUM regularmente:

  .. code-block:: bash

     # VACUUM ANALYZE completo (fazer fora do horário de pico)
     docker exec -ti postgres-postgres-1 psql -U postgres -c "VACUUM ANALYZE;"

- Monitorar bloat de tabelas:

  .. code-block:: bash

     docker exec -ti postgres-postgres-1 psql -U postgres -c "
     SELECT 
       schemaname,
       tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
     FROM pg_tables
     WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
     ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
     LIMIT 10;"

Referências Adicionais
----------------------

- `Documentação oficial de replicação do PostgreSQL <https://www.postgresql.org/docs/current/high-availability.html>`_
- `Patroni para alta disponibilidade <https://patroni.readthedocs.io/>`_
- `PgBouncer <https://www.pgbouncer.org/>`_
- `Monitoramento com Zabbix <https://www.zabbix.com/integrations/postgresql>`_
