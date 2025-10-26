Patroni - Alta Disponibilidade para PostgreSQL
==============================================

O Patroni é um template desenvolvido em Python para garantir alta disponibilidade de clusters PostgreSQL. Ele automatiza o gerenciamento de falhas, eleição de líder e replicação entre os nós do cluster.

Conceitos Fundamentais
----------------------

O que é Patroni?
~~~~~~~~~~~~~~~~

- **Template de HA**: Conjunto de scripts Python que gerenciam automaticamente failover
- **DCS (Distributed Configuration Store)**: Utiliza etcd, Consul ou ZooKeeper para coordenação
- **Gestão automática**: Detecta falhas e promove réplicas automaticamente
- **API REST**: Expõe endpoints para monitoramento e controle

.. warning::
   **Importante**: Apenas o Patroni deve gerenciar o PostgreSQL. Não reinicie o serviço PostgreSQL diretamente - sempre interaja com o ``patronictl`` ou REST API.

Arquitetura do Cluster
~~~~~~~~~~~~~~~~~~~~~

Um cluster Patroni típico consiste em:

- **Leader**: Nó primário que aceita leituras e escritas
- **Replicas**: Nós secundários que replicam dados do líder
- **DCS**: Sistema de coordenação distribuída (etcd, Consul, ZooKeeper)
- **Watchdog**: Opcional, previne split-brain

Standby Cluster (Multi-Datacenter)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Para replicação entre datacenters, utiliza-se a característica **standby cluster**:

- Cluster que não tem nenhum nó PostgreSQL primário em execução
- Não há nenhum membro de leitura/gravação no cluster
- Existe um **líder standby** responsável por replicar de um nó remoto
- Outros standbys replicam em cascata desse líder
- Útil para disaster recovery e replicação entre datacenters

.. note::
   O standby cluster não sabe nada sobre o cluster de origem - pode usar ``restore_command`` em vez de streaming WAL e pode usar um DCS absolutamente independente.

Configuração e Instalação
--------------------------

Pré-requisitos
~~~~~~~~~~~~~~

- PostgreSQL instalado
- DCS configurado (etcd, Consul ou ZooKeeper)
- Python 3.6+
- Rede entre os nós

Docker Compose - Exemplo Básico
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: yaml

   version: '3.8'
   
   networks:
     patroni-net:
       driver: bridge
   
   services:
     etcd:
       image: quay.io/coreos/etcd:latest
       container_name: etcd
       environment:
         - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
         - ETCD_ADVERTISE_CLIENT_URLS=http://etcd:2379
       networks:
         - patroni-net
       ports:
         - "2379:2379"
   
     patroni1:
       image: patroni/patroni:latest
       container_name: patroni1
       environment:
         - PATRONI_NAME=patroni1
         - PATRONI_SCOPE=server1
         - PATRONI_ETCD3_HOSTS=etcd:2379
         - PATRONI_POSTGRESQL_DATA_DIR=/var/lib/postgresql/data
         - PATRONI_POSTGRESQL_LISTEN=0.0.0.0:5432
         - PATRONI_RESTAPI_LISTEN=0.0.0.0:8008
         - PATRONI_SUPERUSER_USERNAME=postgres
         - PATRONI_SUPERUSER_PASSWORD=postgres_password
         - PATRONI_REPLICATION_USERNAME=replicator
         - PATRONI_REPLICATION_PASSWORD=replicator_password
       networks:
         - patroni-net
       ports:
         - "5432:5432"
         - "8008:8008"
       volumes:
         - ./data/patroni1:/var/lib/postgresql/data
       depends_on:
         - etcd
   
     patroni2:
       image: patroni/patroni:latest
       container_name: patroni2
       environment:
         - PATRONI_NAME=patroni2
         - PATRONI_SCOPE=server1
         - PATRONI_ETCD3_HOSTS=etcd:2379
         - PATRONI_POSTGRESQL_DATA_DIR=/var/lib/postgresql/data
         - PATRONI_POSTGRESQL_LISTEN=0.0.0.0:5432
         - PATRONI_RESTAPI_LISTEN=0.0.0.0:8008
         - PATRONI_SUPERUSER_USERNAME=postgres
         - PATRONI_SUPERUSER_PASSWORD=postgres_password
         - PATRONI_REPLICATION_USERNAME=replicator
         - PATRONI_REPLICATION_PASSWORD=replicator_password
       networks:
         - patroni-net
       ports:
         - "5433:5432"
         - "8009:8008"
       volumes:
         - ./data/patroni2:/var/lib/postgresql/data
       depends_on:
         - etcd
   
     patroni3:
       image: patroni/patroni:latest
       container_name: patroni3
       environment:
         - PATRONI_NAME=patroni3
         - PATRONI_SCOPE=server1
         - PATRONI_ETCD3_HOSTS=etcd:2379
         - PATRONI_POSTGRESQL_DATA_DIR=/var/lib/postgresql/data
         - PATRONI_POSTGRESQL_LISTEN=0.0.0.0:5432
         - PATRONI_RESTAPI_LISTEN=0.0.0.0:8008
         - PATRONI_SUPERUSER_USERNAME=postgres
         - PATRONI_SUPERUSER_PASSWORD=postgres_password
         - PATRONI_REPLICATION_USERNAME=replicator
         - PATRONI_REPLICATION_PASSWORD=replicator_password
       networks:
         - patroni-net
       ports:
         - "5434:5432"
         - "8010:8008"
       volumes:
         - ./data/patroni3:/var/lib/postgresql/data
       depends_on:
         - etcd

Iniciando o Cluster
~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Criar diretórios de dados
   mkdir -p data/patroni{1,2,3}
   
   # Iniciar os containers
   docker compose up -d
   
   # Verificar status
   docker compose logs -f

Comandos patronictl
-------------------

Listar Membros do Cluster
~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   docker compose exec patroni1 patronictl list

Saída esperada:

.. code-block:: text

   + Cluster: server1 (7483271224010649624) --------+----+-----------+
   | Member   | Host       | Role    | State    | TL | Lag in MB |
   +----------+------------+---------+----------+----+-----------+
   | patroni1 | 172.23.0.5 | Leader  | running  | 91 |           |
   | patroni2 | 172.23.0.6 | Replica | running  |    |         0 |
   | patroni3 | 172.23.0.3 | Replica | running  |    |         0 |
   +----------+------------+---------+----------+----+-----------+

Executar Queries no Cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Query no líder (-r leader)
   docker compose exec patroni1 patronictl query server1 -r leader \
     -u postgres --password -c "SELECT current_database(), pg_is_in_recovery();" \
     --format pretty -W
   
   # Query em réplica específica (-m membro)
   docker compose exec patroni1 patronictl query server1 -m patroni2 \
     -u postgres --password -c "SELECT * FROM pg_stat_replication;" \
     --format pretty -W
   
   # Monitorar mudanças em tempo real
   docker compose exec patroni1 patronictl query server1 -r leader \
     -u postgres --password -c "SELECT NOW(), count(*) FROM nextcloud;" \
     --format pretty -W

Verificar Versão do PostgreSQL
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   docker compose exec patroni1 patronictl version server1

Mostrar Configuração
~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Mostrar configurações dinâmicas armazenadas no DCS
   docker compose exec patroni1 patronictl show-config server1

Monitoramento via Observação Contínua
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Observar mudanças no cluster a cada 2 segundos
   docker compose exec patroni1 patronictl list -W
   
   # Ou com intervalo customizado
   watch -n 5 'docker compose exec patroni1 patronictl list'

Gerenciamento de Falhas
------------------------

Cenário: Membro com Falha
~~~~~~~~~~~~~~~~~~~~~~~~~~

Quando um membro apresenta problemas:

.. code-block:: text

   + Cluster: server1 (7483271224010649624) --------+----+-----------+
   | Member   | Host       | Role    | State        | TL | Lag in MB |
   +----------+------------+---------+--------------+----+-----------+
   | patroni1 | 172.23.0.5 | Leader  | running      | 91 |           |
   | patroni2 | 172.23.0.6 | Replica | starting     |    |   unknown |
   | patroni3 | 172.23.0.3 | Replica | start failed |    |   unknown |
   +----------+------------+---------+--------------+----+-----------+

Reiniciar Membro com Falha
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Opção 1: Restart agendado
   docker compose exec patroni1 patronictl restart server1 patroni3 \
     --scheduled now --force
   
   # Opção 2: Reinitialize (rebuild completo)
   docker compose exec patroni1 patronictl reinit server1 patroni3

O comando ``reinit`` reconstrói completamente uma instância PostgreSQL standby:

- Remove dados antigos
- Faz novo ``pg_basebackup`` do líder
- Reconfigura replicação

Observar o Progresso
~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Modo watch - atualiza automaticamente
   docker compose exec patroni1 patronictl list -W

Reinitialize de Fonte Específica
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Fazer basebackup diretamente do líder
   docker compose exec patroni1 patronictl reinit server1 patroni2 --from-leader
   
   # Fazer basebackup de outra réplica
   docker compose exec patroni1 patronictl reinit server1 patroni3 --from patroni2

Failover e Switchover
---------------------

Switchover (Planejado)
~~~~~~~~~~~~~~~~~~~~~~

Switchover é uma troca planejada de líder quando o cluster está saudável:

.. code-block:: bash

   # Interativo - pergunta qual será o novo líder
   docker compose exec patroni1 patronictl switchover server1
   
   # Não-interativo - especifica líder e candidato
   docker compose exec patroni1 patronictl switchover server1 \
     --leader patroni1 --candidate patroni2 --force

Casos de uso:

- Manutenção programada no líder atual
- Balanceamento de carga entre datacenters
- Testes de failover

Failover (Não Planejado)
~~~~~~~~~~~~~~~~~~~~~~~~~

Failover é acionado automaticamente quando o líder falha, ou manualmente em emergências:

.. code-block:: bash

   # Failover manual forçado
   docker compose exec patroni1 patronictl failover server1 \
     --candidate patroni2 --force

Cenário: Cluster Sem Líder
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Erro comum: ``This cluster has no leader``

**Solução**: Forçar eleição de um líder

.. code-block:: bash

   # Forçar failover com candidato específico
   docker compose exec patroni1 patronictl failover server1 \
     --candidate patroni2 --force
   
   # Ou deixar o Patroni escolher automaticamente
   docker compose exec patroni1 patronictl failover server1 --force

.. warning::
   Use ``--force`` com cautela. Certifique-se de que não há split-brain no cluster.

Pause e Resume
--------------

Pausar Cluster
~~~~~~~~~~~~~~

Pausar o cluster desabilita o failover automático - útil para manutenção:

.. code-block:: bash

   docker compose exec patroni1 patronictl pause server1

Enquanto pausado:

- Failover automático está desabilitado
- Replicação continua funcionando
- Útil para manutenções que requerem controle manual

Resumir Cluster
~~~~~~~~~~~~~~~

.. code-block:: bash

   # Tira o cluster do modo manutenção e rehabilita failover
   docker compose exec patroni1 patronictl resume server1

Monitoramento
-------------

API REST do Patroni
~~~~~~~~~~~~~~~~~~~

Cada membro Patroni expõe uma API REST na porta 8008:

.. code-block:: bash

   # Verificar status de um membro
   curl http://172.23.0.5:8008/
   
   # Via Docker
   docker compose exec patroni1 curl http://172.23.0.3:8008

Endpoints Úteis
~~~~~~~~~~~~~~~

1. **/** - Status geral do membro

   .. code-block:: bash
   
      curl http://172.23.0.5:8008/

2. **/patroni** - Status do cluster em JSON

   .. code-block:: bash
   
      curl http://172.23.0.5:8008/patroni

3. **/metrics** - Métricas para Prometheus

   .. code-block:: bash
   
      curl http://172.23.0.5:8008/metrics

4. **/health** - Health check (retorna 200 se saudável)

   .. code-block:: bash
   
      curl -f http://172.23.0.5:8008/health && echo "Healthy" || echo "Unhealthy"

5. **/leader** - Retorna 200 apenas se o nó é o líder

   .. code-block:: bash
   
      curl -f http://172.23.0.5:8008/leader

6. **/replica** - Retorna 200 apenas se o nó é uma réplica

   .. code-block:: bash
   
      curl -f http://172.23.0.6:8008/replica

Integração com Prometheus
~~~~~~~~~~~~~~~~~~~~~~~~~~

Adicione ao ``prometheus.yml``:

.. code-block:: yaml

   scrape_configs:
     - job_name: 'patroni'
       static_configs:
         - targets:
           - 'patroni1:8008'
           - 'patroni2:8008'
           - 'patroni3:8008'

Métricas importantes:

- ``patroni_patroni_info`` - Informações sobre o membro
- ``patroni_postgres_running`` - PostgreSQL está rodando (1/0)
- ``patroni_replication_lag`` - Lag de replicação em bytes
- ``patroni_timeline`` - Timeline atual do WAL

Script de Monitoramento
~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   #!/bin/bash
   # monitor-patroni.sh
   
   CLUSTER_NAME="server1"
   ALERT_EMAIL="admin@example.com"
   
   # Verificar se há líder
   LEADER=$(docker compose exec patroni1 patronictl list --format json | \
     jq -r '.[] | select(.Role == "Leader") | .Member')
   
   if [ -z "$LEADER" ]; then
     echo "ALERTA: Cluster $CLUSTER_NAME sem líder!"
     # Enviar notificação
   fi
   
   # Verificar lag de replicação
   docker compose exec patroni1 patronictl list --format json | \
     jq -r '.[] | select(.Role == "Replica") | "\(.Member): \(.["Lag in MB"]) MB"' | \
     while read line; do
       LAG=$(echo $line | awk '{print $2}')
       if [ "$LAG" -gt 100 ]; then
         echo "ALERTA: $line - Lag alto!"
       fi
     done

Backup e Restore
----------------

Backup via pg_dump
~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Fazer dump do banco de dados
   docker compose exec patroni1 pg_dump -U postgres -d nextcloud \
     -F c -b -v -f /tmp/dump.sql
   
   # Copiar dump para o host
   docker cp patroni1:/tmp/dump.sql ./backup_$(date +%Y%m%d).sql

Parâmetros:

- ``-F c``: Formato custom (comprimido)
- ``-b``: Incluir large objects
- ``-v``: Verbose

Restore com pg_restore
~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Copiar dump para o container
   docker cp backup_20251023.sql patroni1:/tmp/dump.sql
   
   # Restaurar banco
   docker compose exec patroni1 pg_restore -d nextcloud -C -j 4 /tmp/dump.sql

Parâmetros:

- ``-C``: Criar banco antes de restaurar
- ``-j 4``: Usar 4 jobs paralelos

.. warning::
   Faça backup de réplicas para não sobrecarregar o líder em produção.

Backup Físico (pg_basebackup)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Via réplica (recomendado)
   docker compose exec patroni2 pg_basebackup -D /backup/base \
     -F tar -z -P -U replicator

Migração: Standalone para Patroni
----------------------------------

Para converter uma instância PostgreSQL standalone em cluster Patroni, consulte a `documentação oficial <https://patroni.readthedocs.io/en/latest/existing_data.html#existing-data>`_.

Resumo do Processo
~~~~~~~~~~~~~~~~~~

1. **Criar usuários Patroni**:

   .. code-block:: sql
   
      -- Superuser do Patroni
      CREATE USER patroni_user WITH SUPERUSER ENCRYPTED PASSWORD 'senha';
      
      -- Usuário de replicação
      CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'senha';

2. **Configurar pg_hba.conf**:

   .. code-block:: text
   
      host    all             patroni_user    127.0.0.1/32    scram-sha-256
      host    replication     replicator      all             scram-sha-256

3. **Parar PostgreSQL standalone**:

   .. code-block:: bash
   
      systemctl stop postgresql

4. **Mover dados para diretório Patroni**:

   .. code-block:: bash
   
      mv /var/lib/postgresql/14/main /data/patroni/

5. **Iniciar Patroni** apontando para o diretório de dados existente

6. **Adicionar réplicas** usando ``pg_basebackup``

Solução de Problemas
--------------------

Membro Não Se Conecta ao DCS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Verificar conectividade com etcd
   docker compose exec patroni1 curl http://etcd:2379/health
   
   # Verificar logs
   docker compose logs patroni1 | grep -i etcd

Split-Brain
~~~~~~~~~~~

Sintoma: Dois membros se consideram líderes simultaneamente.

**Prevenção**: Configure watchdog

**Solução**:

1. Parar um dos líderes manualmente
2. Fazer ``reinit`` do membro que estava errado
3. Verificar configuração de DCS e rede

Replicação Atrasada
~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Verificar lag detalhado
   docker compose exec patroni1 patronictl list
   
   # Ver configurações de replicação
   docker compose exec patroni1 patronictl show-config server1 | grep -i wal

Possíveis causas:

- Rede lenta entre nós
- Líder sobrecarregado
- Disco lento na réplica
- ``max_wal_senders`` muito baixo

Cluster Não Inicializa
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Verificar logs de todos os membros
   docker compose logs patroni1 patroni2 patroni3
   
   # Verificar DCS
   docker compose exec patroni1 patronictl list

Possíveis problemas:

- DCS (etcd) não está rodando
- Configuração incorreta de rede
- Dados corrompidos no diretório PostgreSQL

Boas Práticas
-------------

Configuração de Produção
~~~~~~~~~~~~~~~~~~~~~~~~

1. **Use watchdog** para prevenir split-brain
2. **Configure synchronous_mode** para zero data loss
3. **Monitore métricas** com Prometheus/Grafana
4. **Faça backups** regulares das réplicas
5. **Teste failover** regularmente em ambiente de staging
6. **Configure alertas** para lag de replicação
7. **Use VPN/SSL** para comunicação entre datacenters

Parâmetros Recomendados
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: yaml

   # patroni.yml
   bootstrap:
     dcs:
       ttl: 30
       loop_wait: 10
       retry_timeout: 10
       maximum_lag_on_failover: 1048576  # 1MB
       
     postgresql:
       parameters:
         max_connections: 100
         shared_buffers: 256MB
         effective_cache_size: 1GB
         wal_level: replica
         max_wal_senders: 10
         max_replication_slots: 10
         hot_standby: on

Segurança
~~~~~~~~~

- Use senhas fortes para usuários PostgreSQL
- Configure SSL/TLS para conexões
- Limite acesso à API REST com firewall
- Use SCRAM-SHA-256 para autenticação
- Mantenha Patroni e PostgreSQL atualizados

Referências
-----------

- `Documentação oficial do Patroni <https://patroni.readthedocs.io/>`_
- `patronictl reference <https://patroni.readthedocs.io/en/latest/patronictl.html>`_
- `Configuração de HA Multi-Datacenter <https://patroni.readthedocs.io/en/latest/ha_multi_dc.html>`_
- `Standby Cluster <https://patroni.readthedocs.io/en/latest/standby_cluster.html>`_
- `REST API <https://patroni.readthedocs.io/en/latest/rest_api.html>`_
