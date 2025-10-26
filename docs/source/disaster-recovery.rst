Recuperação de Desastres (Disaster Recovery)
============================================

Este documento descreve os procedimentos para recuperação do Nextcloud em caso de desastres e falhas críticas em ambientes distribuídos.

Visão Geral
-----------

Tipos de Desastres
~~~~~~~~~~~~~~~~~~

1. **Falha de Datacenter Completo**
   - Perda total de um datacenter (incêndio, inundação, falta de energia prolongada)
   - Perda de conectividade de rede

2. **Falha de Componentes Críticos**
   - Cluster de banco de dados inoperante
   - Sistema de armazenamento corrompido
   - Perda do servidor proxy/load balancer

3. **Corrupção de Dados**
   - Dados corrompidos por ransomware
   - Deleção acidental de dados
   - Corrupção de base de dados

4. **Desastres Naturais**
   - Eventos que afetam toda uma região geográfica

Objetivos de Recuperação
~~~~~~~~~~~~~~~~~~~~~~~~

- **RTO (Recovery Time Objective)**: Tempo máximo aceitável de inatividade
- **RPO (Recovery Point Objective)**: Quantidade máxima aceitável de perda de dados

.. note::
   Para ambientes de produção críticos, recomenda-se:
   
   - RTO: < 4 horas
   - RPO: < 1 hora

Infraestrutura de Disaster Recovery
------------------------------------

Topologia Recomendada
~~~~~~~~~~~~~~~~~~~~~

Para DR adequado, deve-se ter:

- **Datacenter Primário**: Cluster ativo com todos os serviços
- **Datacenter Secundário**: Standby cluster em outra região geográfica
- **Backup Offsite**: Armazenamento em nuvem (S3, Wasabi, MinIO remoto)

Arquitetura Multi-Datacenter
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: text

   ┌─────────────────────────────────┐        ┌─────────────────────────────────┐
   │     Datacenter 1 (Primário)     │        │    Datacenter 2 (Secundário)    │
   │                                 │        │                                 │
   │  ┌──────────┐  ┌─────────────┐ │        │  ┌──────────┐  ┌─────────────┐ │
   │  │ Nextcloud│  │  PostgreSQL │ │◄──────►│  │ Nextcloud│  │  PostgreSQL │ │
   │  │  (Ativo) │  │   (Leader)  │ │Replica │  │ (Standby)│  │  (Replica)  │ │
   │  └──────────┘  └─────────────┘ │        │  └──────────┘  └─────────────┘ │
   │                                 │        │                                 │
   │  ┌─────────────────────────┐   │        │  ┌─────────────────────────┐   │
   │  │  GlusterFS/GarageS3     │◄──┼────────┼─►│  GlusterFS/GarageS3     │   │
   │  │  (Sincronização)        │   │        │  │  (Sincronização)        │   │
   │  └─────────────────────────┘   │        │  └─────────────────────────┘   │
   └─────────────────────────────────┘        └─────────────────────────────────┘
                    │                                           │
                    │                                           │
                    └───────────────┬───────────────────────────┘
                                    │
                                    ▼
                          ┌─────────────────┐
                          │  Backup Offsite │
                          │  (S3/Wasabi)    │
                          └─────────────────┘

Componentes Essenciais para DR
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. **Replicação de Banco de Dados**
   
   - PostgreSQL com replicação streaming
   - Patroni para failover automático
   - WAL archiving para PITR

2. **Sincronização de Arquivos**
   
   - GlusterFS com replicação geográfica
   - GarageS3 para armazenamento distribuído
   - Rsync/Syncthing para sincronização adicional

3. **Backups Incrementais**
   
   - Duplicati configurado para múltiplos destinos
   - Snapshots de volumes
   - Dumps de banco de dados regulares

4. **Monitoramento**
   
   - Zabbix monitorando todos os componentes
   - Alertas configurados para eventos críticos
   - Dashboard de status de DR

Procedimentos de Recuperação
-----------------------------

Cenário 1: Falha Total do Datacenter Primário
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Sintomas:**
- Datacenter 1 completamente inacessível
- Todos os serviços inoperantes
- Perda de conectividade total

**Passos de Recuperação:**

1. **Avaliar a Situação (0-15 min)**

   .. code-block:: bash
   
      # Verificar status do datacenter primário
      ping servidor1.datacenter1.com
      
      # Testar conectividade SSH
      ssh user@servidor1.datacenter1.com
      
      # Verificar status via monitoramento
      curl https://monitoramento.empresa.com/status/dc1

2. **Ativar Datacenter Secundário (15-30 min)**

Cenário 1A: Com Patroni (Failover Automático)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. note::
   Se você está usando Patroni com Standby Cluster, o failover pode ser automático ou manual.

a. **Verificar Status do Cluster Patroni**:

   .. code-block:: bash
   
      # Verificar se failover automático já ocorreu
      docker compose exec patroni1-dc2 patronictl list
      
      # Verificar se DC2 possui líder
      docker compose exec patroni1-dc2 patronictl list | grep Leader

b. **Promover Standby Cluster (se necessário)**:

   .. code-block:: bash
   
      # Se o cluster DC2 ainda estiver em modo standby
      # Desabilitar standby mode
      docker compose exec patroni1-dc2 patronictl edit-config cluster-dc2
      
      # Remover a configuração standby_cluster na seção DCS
      # Ou via REST API:
      curl -X PATCH http://patroni1-dc2:8008/config \
        -d '{"standby_cluster": null}'
      
      # Aguardar eleição de líder
      docker compose exec patroni1-dc2 patronictl list -W
      
      # Ou forçar failover se necessário
      docker compose exec patroni1-dc2 patronictl failover cluster-dc2 \
        --candidate patroni1-dc2 --force
      
      # Verificar novo líder
      docker compose exec patroni1-dc2 patronictl list

c. **Validar Cluster**:

   .. code-block:: bash
   
      # Verificar que há um líder ativo
      docker compose exec patroni1-dc2 patronictl list
      
      # Testar conexão ao banco
      docker compose exec patroni1-dc2 psql -U postgres -c "SELECT pg_is_in_recovery();"
      # Deve retornar 'f' (false) se for o líder

Cenário 1B: PostgreSQL Nativo (Promoção Manual)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. warning::
   Sem Patroni, você precisará promover manualmente a réplica PostgreSQL.

a. **Promover Réplica para Primário**:

   .. code-block:: bash
   
      # Método 1: Usando pg_ctl
      docker compose exec postgres-replica pg_ctl promote -D /var/lib/postgresql/data
      
      # Método 2: Usando SQL (PostgreSQL 12+)
      docker compose exec postgres-replica psql -U postgres -c "SELECT pg_promote();"
      
      # Método 3: Criar arquivo trigger (método antigo)
      docker compose exec postgres-replica touch /var/lib/postgresql/data/promote.trigger

b. **Verificar Promoção**:

   .. code-block:: bash
   
      # Verificar se não está mais em recovery
      docker compose exec postgres-replica psql -U postgres -c "SELECT pg_is_in_recovery();"
      # Deve retornar 'f' (false)
      
      # Verificar logs
      docker compose logs postgres-replica | grep -i promote

c. **Atualizar Configurações**:

   .. code-block:: bash
   
      # Remover configurações de réplica no postgresql.auto.conf
      docker compose exec postgres-replica bash -c "cat > /var/lib/postgresql/data/postgresql.auto.conf << EOF
      # Agora é primário - sem configurações de replica
      EOF"
      
      # Reiniciar para aplicar
      docker compose restart postgres-replica

d. **Reconfigurar pg_hba.conf para Aceitar Réplicas**:

   .. code-block:: bash
   
      # Adicionar entrada para futuras réplicas
      docker compose exec postgres-replica bash -c "cat >> /var/lib/postgresql/data/pg_hba.conf << EOF
      host    replication     replicator      IP_FUTURA_REPLICA/32    scram-sha-256
      EOF"
      
      # Reload configuração
      docker compose exec postgres-replica psql -U postgres -c "SELECT pg_reload_conf();"

Próximos Passos (Comum para Patroni e PostgreSQL Nativo)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

b. **Ativar Nextcloud no DC2**:
   
      .. code-block:: bash
      
         # Atualizar DNS para apontar para DC2
         # Via API do provedor DNS ou manualmente
         
         # Iniciar containers Nextcloud
         cd /home/nextcloud/dc2
         docker compose up -d
         
         # Verificar logs
         docker compose logs -f nextcloud

   c. **Validar Armazenamento**:
   
      .. code-block:: bash
      
         # Verificar montagem GlusterFS
         df -h | grep gluster
         
         # Testar leitura/escrita
         touch /mnt/gluster/test-dr.txt
         ls -l /mnt/gluster/test-dr.txt
         rm /mnt/gluster/test-dr.txt

3. **Atualizar DNS (30-45 min)**

   .. code-block:: bash
   
      # Exemplo com CloudFlare API
      curl -X PUT "https://api.cloudflare.com/client/v4/zones/{zone_id}/dns_records/{record_id}" \
        -H "Authorization: Bearer {api_token}" \
        -H "Content-Type: application/json" \
        --data '{
          "type": "A",
          "name": "nextcloud.empresa.com",
          "content": "IP_DC2",
          "ttl": 300,
          "proxied": false
        }'
   
   .. warning::
      Aguarde o TTL do DNS expirar (geralmente 5-15 minutos) antes de considerar a migração completa.

4. **Validar Serviços (45-60 min)**

   .. code-block:: bash
   
      # Testar acesso web
      curl -I https://nextcloud.empresa.com
      
      # Verificar banco de dados
      docker compose exec patroni1-dc2 psql -U postgres -d nextcloud -c "SELECT count(*) FROM oc_users;"
      
      # Testar login de usuário
      curl -u usuario:senha https://nextcloud.empresa.com/ocs/v1.php/cloud/capabilities
      
      # Verificar logs de erro
      docker compose logs nextcloud | grep -i error

5. **Comunicar Usuários (60 min)**

   - Notificar equipe que o sistema está operacional
   - Informar sobre possível perda de dados (conforme RPO)
   - Documentar o incidente

**Checklist de Validação:**

.. code-block:: text

   ☐ PostgreSQL está rodando e aceitando conexões
   ☐ Patroni mostra DC2 como líder
   ☐ Nextcloud está acessível via web
   ☐ Usuários conseguem fazer login
   ☐ Upload/download de arquivos funcionando
   ☐ DNS atualizado e propagado
   ☐ Certificado SSL válido
   ☐ Monitoramento do DC2 ativo

Cenário 2: Falha do Cluster PostgreSQL
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Sintomas:**
- Erros de conexão ao banco de dados
- Cluster sem líder (Patroni) ou serviço PostgreSQL parado (nativo)
- Aplicação Nextcloud com timeout

Cenário 2A: Falha com Patroni
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Diagnóstico:**

.. code-block:: bash

   # Verificar status do cluster
   docker compose exec patroni1 patronictl list
   
   # Verificar logs do Patroni
   docker compose logs patroni1 patroni2 patroni3 | tail -100
   
   # Testar conexão direta
   docker compose exec patroni1 psql -U postgres -c "SELECT 1;"

**Recuperação:**

1. **Cluster sem líder mas dados intactos**:

   .. code-block:: bash
   
      # Forçar eleição
      docker compose exec patroni1 patronictl failover cluster-name \
        --candidate patroni1 --force
      
      # Verificar
      docker compose exec patroni1 patronictl list

2. **Todos os nós Patroni com falha**:

   .. code-block:: bash
   
      # Parar todos os containers
      docker compose down
      
      # Identificar nó com dados mais recentes
      # Verificar LSN (Log Sequence Number)
      cat /data/patroni1/data/pg_control | grep "Latest checkpoint's REDO location"
      cat /data/patroni2/data/pg_control | grep "Latest checkpoint's REDO location"
      cat /data/patroni3/data/pg_control | grep "Latest checkpoint's REDO location"
      
      # Iniciar o nó com LSN mais alto primeiro
      docker compose up -d patroni1
      
      # Aguardar ele se tornar líder
      docker compose exec patroni1 patronictl list
      
      # Iniciar outros nós
      docker compose up -d patroni2 patroni3

3. **Corrupção de dados com Patroni - Restaurar de backup**:

   .. code-block:: bash
   
      # Parar cluster
      docker compose down
      
      # Limpar dados corrompidos
      rm -rf /data/patroni*/data/*
      
      # Restaurar backup mais recente
      tar -xzf /backup/postgres-basebackup-20251026.tar.gz -C /data/patroni1/data/
      
      # Restaurar WAL archives para PITR
      cp /backup/wal_archive/* /data/patroni1/data/pg_wal/
      
      # Configurar recovery
      cat > /data/patroni1/data/recovery.signal << EOF
      restore_command = 'cp /backup/wal_archive/%f %p'
      recovery_target_time = '2025-10-26 12:00:00'
      EOF
      
      # Iniciar primeiro nó
      docker compose up -d patroni1
      
      # Após recovery, recriar réplicas
      docker compose exec patroni1 patronictl reinit cluster patroni2 --force
      docker compose exec patroni1 patronictl reinit cluster patroni3 --force

Cenário 2B: Falha com PostgreSQL Nativo
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Diagnóstico:**

.. code-block:: bash

   # Verificar se o PostgreSQL está rodando
   docker compose ps postgres
   
   # Verificar logs
   docker compose logs postgres | tail -100
   
   # Verificar status do processo
   docker compose exec postgres pg_isready -U postgres

**Recuperação:**

1. **PostgreSQL primário não inicia**:

   .. code-block:: bash
   
      # Verificar logs de erro
      docker compose logs postgres | grep -i "fatal\|error"
      
      # Verificar integridade dos dados
      docker compose exec postgres pg_controldata /var/lib/postgresql/data
      
      # Se houver corrupção menor, tentar recovery
      docker compose exec postgres pg_resetwal -f /var/lib/postgresql/data
      
      # Reiniciar
      docker compose restart postgres

2. **Réplica desconectada do primário**:

   .. code-block:: bash
   
      # No primário, verificar slots de replicação
      docker compose exec postgres psql -U postgres -c "
        SELECT slot_name, active, restart_lsn 
        FROM pg_replication_slots;"
      
      # Na réplica, verificar status
      docker compose exec postgres-replica psql -U postgres -c "
        SELECT pg_last_wal_receive_lsn(), 
               pg_last_wal_replay_lsn();"
      
      # Se LSN não avança, reconstruir réplica
      docker compose exec postgres-replica pg_ctl stop -D /var/lib/postgresql/data
      rm -rf /volumes/postgres-replica/data/*
      
      # Fazer novo pg_basebackup
      docker run --rm -v ./volumes/postgres-replica/data:/var/lib/postgresql/data postgres:16-alpine \
        bash -c "PGPASSWORD=\$REPLICATION_PASSWORD pg_basebackup \
          -h postgres -U replicator \
          -D /var/lib/postgresql/data -P -R -v --wal-method=stream"
      
      # Reiniciar réplica
      docker compose up -d postgres-replica

3. **Primário com falha - promover réplica**:

   .. code-block:: bash
   
      # Parar o primário com problema
      docker compose stop postgres
      
      # Promover réplica (ver Cenário 1B acima)
      docker compose exec postgres-replica pg_ctl promote -D /var/lib/postgresql/data
      
      # Verificar promoção
      docker compose exec postgres-replica psql -U postgres -c "SELECT pg_is_in_recovery();"
      
      # Atualizar aplicação Nextcloud para apontar para novo host
      # Editar config.php ou variáveis de ambiente
      docker compose exec nextcloud vi /var/www/html/config/config.php
      # Mudar 'dbhost' para o IP/nome da antiga réplica
      
      # Reiniciar Nextcloud
      docker compose restart nextcloud

4. **Corrupção de dados - Restaurar de backup**:

   .. code-block:: bash
   
      # Parar PostgreSQL
      docker compose down postgres
      
      # Limpar dados corrompidos
      rm -rf /volumes/postgres/data/*
      
      # Restaurar backup base
      tar -xzf /backup/postgres-basebackup-20251026.tar.gz -C /volumes/postgres/data/
      
      # Configurar recovery para PITR
      cat > /volumes/postgres/data/recovery.signal << EOF
      restore_command = 'cp /backup/wal_archive/%f %p'
      recovery_target_time = '2025-10-26 12:00:00'
      recovery_target_action = 'promote'
      EOF
      
      # Iniciar PostgreSQL
      docker compose up -d postgres
      
      # Monitorar recovery
      docker compose logs -f postgres | grep recovery
      
      # Após recovery, recriar réplicas
      # Na réplica, fazer novo pg_basebackup do primário restaurado
      docker compose exec postgres-replica pg_ctl stop
      rm -rf /volumes/postgres-replica/data/*
      
      docker run --rm -v ./volumes/postgres-replica/data:/var/lib/postgresql/data postgres:16-alpine \
        bash -c "PGPASSWORD=\$REPLICATION_PASSWORD pg_basebackup \
          -h postgres -U replicator \
          -D /var/lib/postgresql/data -P -R -v --wal-method=stream"
      
      docker compose up -d postgres-replica

Cenário 3: Corrupção de Armazenamento (Ransomware)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Sintomas:**
- Arquivos criptografados ou modificados
- Notas de resgate nos diretórios
- Usuários reportando arquivos inacessíveis

**Ações Imediatas:**

.. code-block:: bash

   # PASSO 1: ISOLAR IMEDIATAMENTE
   # Desconectar sistema da rede
   docker compose down
   
   # Desmontar volumes de armazenamento
   umount /mnt/gluster
   
   # Bloquear acesso na rede
   iptables -A INPUT -j DROP
   iptables -A OUTPUT -j DROP

**Recuperação:**

1. **Avaliar Extensão do Dano**:

   .. code-block:: bash
   
      # Verificar arquivos modificados recentemente
      find /mnt/gluster-backup -type f -mtime -1 -ls
      
      # Verificar extensões suspeitas
      find /mnt/gluster-backup -name "*.encrypted" -o -name "*.locked"
      
      # Verificar tamanho do dano
      du -sh /mnt/gluster-backup

2. **Restaurar de Backup Limpo**:

   .. code-block:: bash
   
      # Identificar último backup antes do ataque
      duplicati-cli list s3://bucket-backup/nextcloud/
      
      # Restaurar para novo volume
      duplicati-cli restore s3://bucket-backup/nextcloud/ \
        --restore-path=/mnt/recovery/ \
        --version=2025-10-25T18:00:00
      
      # Verificar integridade
      ls -lR /mnt/recovery/
      
      # Montar novo volume
      mount -t glusterfs server:/volume /mnt/gluster

3. **Restaurar Banco de Dados**:

   .. code-block:: bash
   
      # Restaurar dump de antes do ataque
      docker compose exec patroni1 pg_restore \
        -d nextcloud -C -j 4 /backup/nextcloud-20251025.sql
      
      # Verificar dados
      docker compose exec patroni1 psql -U postgres -d nextcloud \
        -c "SELECT COUNT(*) FROM oc_filecache;"

4. **Reativar Serviços**:

   .. code-block:: bash
   
      # Remover bloqueios de rede
      iptables -F
      
      # Iniciar Nextcloud
      docker compose up -d
      
      # Executar scan de arquivos
      docker compose exec nextcloud php occ files:scan --all

5. **Análise Forense**:

   - Identificar vetor de ataque
   - Revisar logs de acesso
   - Verificar contas comprometidas
   - Implementar melhorias de segurança

Cenário 4: Perda de Sincronização GlusterFS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Sintomas:**
- Dados divergentes entre bricks
- Erros de split-brain
- Volume não monta

**Diagnóstico:**

.. code-block:: bash

   # Verificar status do volume
   gluster volume status volume-nextcloud
   
   # Verificar split-brain
   gluster volume heal volume-nextcloud info split-brain
   
   # Ver arquivos com problemas
   gluster volume heal volume-nextcloud info

**Recuperação de Split-Brain:**

.. code-block:: bash

   # Identificar brick com dados corretos
   ls -l /mnt/brick1/path/to/file
   ls -l /mnt/brick2/path/to/file
   
   # Resolver split-brain escolhendo fonte
   gluster volume heal volume-nextcloud split-brain source-brick \
     server1:/brick1 /path/to/file
   
   # Ou escolher automaticamente o maior
   gluster volume heal volume-nextcloud split-brain latest-mtime /path/to/file
   
   # Iniciar heal
   gluster volume heal volume-nextcloud

**Reconstruir Brick Corrompido:**

.. code-block:: bash

   # Remover brick problemático
   gluster volume remove-brick volume-nextcloud replica 2 \
     server2:/brick2 force
   
   # Limpar dados corrompidos
   rm -rf /brick2/*
   
   # Adicionar brick novamente
   gluster volume add-brick volume-nextcloud replica 3 \
     server2:/brick2
   
   # Rebalancear
   gluster volume rebalance volume-nextcloud start

Backups e Restauração
---------------------

Estratégia de Backup 3-2-1
~~~~~~~~~~~~~~~~~~~~~~~~~~

- **3** cópias dos dados
- **2** tipos diferentes de mídia
- **1** cópia offsite

Tipos de Backup
~~~~~~~~~~~~~~~

1. **Backup Completo Semanal**

   .. code-block:: bash
   
      #!/bin/bash
      # backup-full.sh
      
      DATE=$(date +%Y%m%d)
      BACKUP_DIR="/backup/full"
      
      # Backup PostgreSQL
      docker compose exec patroni2 pg_basebackup \
        -D $BACKUP_DIR/postgres-$DATE -F tar -z -P
      
      # Backup arquivos Nextcloud
      tar -czf $BACKUP_DIR/nextcloud-data-$DATE.tar.gz \
        /mnt/gluster/nextcloud/
      
      # Upload para S3
      aws s3 sync $BACKUP_DIR/ s3://backup-nextcloud/full/
      
      # Reter apenas últimos 4 backups completos
      find $BACKUP_DIR -name "*.tar.gz" -mtime +28 -delete

2. **Backup Incremental Diário**

   .. code-block:: bash
   
      #!/bin/bash
      # backup-incremental.sh
      
      DATE=$(date +%Y%m%d)
      
      # Dump SQL incremental
      docker compose exec patroni2 pg_dump -U postgres -d nextcloud \
        -F c -f /backup/incremental/nextcloud-$DATE.dump
      
      # Rsync de arquivos modificados
      rsync -av --link-dest=/backup/full/latest/ \
        /mnt/gluster/nextcloud/ \
        /backup/incremental/$DATE/
      
      # Upload para S3
      aws s3 sync /backup/incremental/$DATE/ \
        s3://backup-nextcloud/incremental/$DATE/

3. **Backup Contínuo de WAL**

   .. code-block:: bash
   
      # Já configurado no postgresql.conf
      archive_mode = on
      archive_command = 'aws s3 cp %p s3://backup-nextcloud/wal/%f'

Restauração Completa do Sistema
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Cenário: Perda total - reconstruir do zero**

1. **Preparar Nova Infraestrutura**:

   .. code-block:: bash
   
      # Provisionar novos servidores
      # Instalar Docker, Docker Compose
      # Configurar rede e DNS

2. **Restaurar PostgreSQL**:

   .. code-block:: bash
   
      # Download do backup
      aws s3 sync s3://backup-nextcloud/full/ /restore/
      
      # Extrair backup
      tar -xzf /restore/postgres-20251025/base.tar.gz -C /data/patroni1/data/
      
      # Configurar recovery para PITR
      cat > /data/patroni1/data/recovery.signal << EOF
      restore_command = 'aws s3 cp s3://backup-nextcloud/wal/%f %p'
      recovery_target_time = '2025-10-26 10:00:00'
      recovery_target_action = 'promote'
      EOF
      
      # Iniciar PostgreSQL
      docker compose up -d patroni1

3. **Restaurar Arquivos**:

   .. code-block:: bash
   
      # Restaurar dados do Nextcloud
      tar -xzf /restore/nextcloud-data-20251025.tar.gz -C /mnt/gluster/
      
      # Ajustar permissões
      chown -R www-data:www-data /mnt/gluster/nextcloud/

4. **Configurar e Iniciar Nextcloud**:

   .. code-block:: bash
   
      # Restaurar configuração
      cp /restore/nextcloud-config.php /mnt/gluster/nextcloud/config/config.php
      
      # Iniciar containers
      docker compose up -d nextcloud
      
      # Executar manutenção
      docker compose exec nextcloud php occ maintenance:mode --off
      docker compose exec nextcloud php occ files:scan --all

5. **Validar Restauração**:

   .. code-block:: bash
   
      # Testar acesso
      curl -I https://nextcloud.empresa.com
      
      # Verificar usuários
      docker compose exec nextcloud php occ user:list
      
      # Verificar arquivos
      docker compose exec nextcloud php occ files:scan --all

Testes de Disaster Recovery
----------------------------

Cronograma de Testes
~~~~~~~~~~~~~~~~~~~~

- **Teste Completo**: Trimestralmente
- **Teste de Failover**: Mensalmente
- **Teste de Backup/Restore**: Semanalmente
- **Validação de Monitoramento**: Diariamente

Homologação de Testes - PostgreSQL Nativo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Este documento detalha cenários testados em ambiente com replicação manual PostgreSQL, incluindo tempo de execução de cada etapa e problemas encontrados.

**Legenda:**

- **S1** = Servidor 1 (Datacenter Primário)
- **S2** = Servidor 2 (Datacenter Secundário)

Cenário de Teste 1: Failover Sem Certificado Wildcard
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Configuração do Ambiente:**

- Certificados SSL específicos por servidor (NÃO wildcard)
- Projetos replicados: ``nginx-proxy``, ``nextcloud-docker``, ``postgres``
- Dados replicados: Nextcloud (via GlusterFS), PostgreSQL (streaming replication)
- **S1 Ativo**: nginx-proxy, nextcloud, postgres (primário)
- **S2 Ativo**: nextcloud (standby), postgres (réplica)
- **S2 Inativo**: nginx-proxy (será iniciado após failover)

**Procedimento de Teste:**

1. **Simular Falha do S1** (Tempo: 1 min)

   .. code-block:: bash
   
      # No S1 - Desligar servidor para simular falha
      sudo shutdown -h now
      
      # Ou parar serviços para teste controlado
      docker compose down

2. **Atualizar DNS** (Tempo: 2-5 min)

   .. code-block:: bash
   
      # Atualizar entrada tipo A no provedor DNS
      # Exemplo: lugar.librecode.coop
      # DE: 189.39.218.225 (S1)
      # PARA: 187.57.255.128 (S2)
      
      # Via CloudFlare API (exemplo)
      curl -X PUT "https://api.cloudflare.com/client/v4/zones/{zone_id}/dns_records/{record_id}" \
        -H "Authorization: Bearer {api_token}" \
        -H "Content-Type: application/json" \
        --data '{
          "type": "A",
          "name": "lugar.librecode.coop",
          "content": "187.57.255.128",
          "ttl": 300
        }'
   
   .. note::
      Aguardar TTL do DNS expirar (geralmente 5-15 minutos) antes de validar acesso.

3. **Promover PostgreSQL Réplica para Primário no S2** (Tempo: 30 seg - 2 min)

   .. code-block:: bash
   
      # Método 1: Via pg_ctl (dentro do container)
      docker compose exec -u postgres postgres pg_ctl promote -D /var/lib/postgresql/data
      
      # Método 2: Via SQL (PostgreSQL 12+)
      docker compose exec postgres psql -U postgres -c "SELECT pg_promote();"
      
      # Verificar promoção bem-sucedida
      docker compose exec postgres psql -U postgres -c "SELECT pg_is_in_recovery();"
      # Deve retornar: f (false = é primário agora)
      
      # Verificar logs de promoção
      docker compose logs postgres | grep -i "promote\|primary"

4. **Inicializar nginx-proxy no S2** (Tempo: 1-2 min)

   .. code-block:: bash
   
      # No S2, iniciar nginx-proxy
      cd /caminho/para/nginx-proxy
      docker compose up -d
      
      # Verificar se iniciou corretamente
      docker compose ps
      docker compose logs nginx-proxy
      
      # Verificar certificação SSL
      curl -I https://lugar.librecode.coop

5. **Validar Nextcloud** (Tempo: 2-5 min)

   .. code-block:: bash
   
      # Nextcloud já deve estar em execução no S2
      docker compose ps nextcloud
      
      # Testar acesso web
      curl -I https://lugar.librecode.coop
      
      # Testar login
      curl -u usuario:senha https://lugar.librecode.coop/ocs/v1.php/cloud/capabilities
      
      # Verificar conexão com banco de dados
      docker compose exec nextcloud php occ status

**Tempo Total Estimado: 10-20 minutos**

.. warning::
   O tempo pode variar dependendo do TTL do DNS e da propagação da alteração.

Problemas Encontrados e Resoluções
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1. **Problema: Nginx-proxy não sincronizado**

   **Sintoma:**
   
   .. code-block:: text
   
      Certificado SSL inválido ou não encontrado no S2
   
   **Causa:**
   
   - Certificados SSL não foram replicados entre servidores
   - Configurações do nginx-proxy não estão sincronizadas
   
   **Solução:**
   
   .. code-block:: bash
   
      # Sincronizar certificados e configurações do nginx-proxy
      # No S2, antes do teste:
      rsync -avz S1:/caminho/nginx-proxy/certs/ /caminho/nginx-proxy/certs/
      rsync -avz S1:/caminho/nginx-proxy/conf.d/ /caminho/nginx-proxy/conf.d/
      rsync -avz S1:/caminho/nginx-proxy/vhost.d/ /caminho/nginx-proxy/vhost.d/
      
      # Ou configurar sincronização automática com GlusterFS
      # Montar volume compartilhado para nginx-proxy
   
   **Prevenção:**
   
   - Usar certificados wildcard (ver Cenário 2)
   - Manter nginx-proxy em volume GlusterFS replicado
   - Configurar cron job para sync diário:
   
   .. code-block:: bash
   
      # Adicionar ao crontab do S2
      0 2 * * * rsync -avz S1:/caminho/nginx-proxy/ /caminho/nginx-proxy/

2. **Problema: Volume GlusterFS inacessível**

   **Sintoma:**
   
   .. code-block:: text
   
      ls: cannot access '/mnt/nextcloud-data': Transport endpoint is not connected
   
   **Causa:**
   
   - GlusterFS não consegue se conectar aos bricks do S1
   - Split-brain no volume
   - Processo glusterd parado
   
   **Solução Imediata:**
   
   .. code-block:: bash
   
      # No S2, remontar volume GlusterFS
      umount /mnt/nextcloud-data
      
      # Verificar status do GlusterFS
      systemctl status glusterd
      
      # Se necessário, reiniciar glusterd
      systemctl restart glusterd
      
      # Verificar peers disponíveis
      gluster peer status
      
      # Remontar volume (somente brick local do S2)
      mount -t glusterfs localhost:/nextcloud-volume /mnt/nextcloud-data
      
      # Verificar montagem
      df -h | grep gluster
      ls -l /mnt/nextcloud-data
   
   **Solução Permanente:**
   
   .. code-block:: bash
   
      # Configurar volume para operar com apenas 1 brick
      gluster volume set nextcloud-volume cluster.quorum-type none
      
      # Atualizar /etc/fstab para usar localhost
      # DE: server1:/nextcloud-volume /mnt/nextcloud-data glusterfs defaults 0 0
      # PARA: localhost:/nextcloud-volume /mnt/nextcloud-data glusterfs defaults,_netdev 0 0

3. **Problema: Nextcloud com erro de configuração**

   **Sintoma:**
   
   .. code-block:: text
   
      Configuration was not read or initialized correctly, 
      not overwriting /var/www/html/config/config.php
      
      Your data directory is invalid.
   
   **Causa:**
   
   - Volume GlusterFS inacessível (ver problema anterior)
   - Permissões incorretas no diretório de dados
   - config.php corrompido ou inacessível
   
   **Solução:**
   
   .. code-block:: bash
   
      # 1. Verificar montagem do volume
      df -h | grep nextcloud-data
      ls -la /mnt/nextcloud-data/
      
      # 2. Verificar permissões
      ls -ld /mnt/nextcloud-data/
      # Deve ser: drwxr-xr-x www-data www-data
      
      # 3. Corrigir permissões se necessário
      docker compose exec nextcloud chown -R www-data:www-data /var/www/html/data
      
      # 4. Verificar config.php
      docker compose exec nextcloud cat /var/www/html/config/config.php
      
      # 5. Recriar container se necessário
      docker compose up -d nextcloud --force-recreate
      
      # 6. Executar manutenção
      docker compose exec nextcloud php occ maintenance:mode --off
      docker compose exec nextcloud php occ files:scan --all

4. **Problema: Conexão ao banco de dados após promoção**

   **Sintoma:**
   
   .. code-block:: text
   
      SQLSTATE[08006] [7] could not connect to server
   
   **Causa:**
   
   - Nextcloud ainda tentando conectar ao host antigo do PostgreSQL
   - config.php não atualizado para novo primário
   
   **Solução:**
   
   .. code-block:: bash
   
      # Verificar configuração atual
      docker compose exec nextcloud php -r "require '/var/www/html/config/config.php'; print_r(\$CONFIG['dbhost']);"
      
      # Se ainda apontar para S1, atualizar
      docker compose exec nextcloud sed -i "s/189.39.218.225/187.57.255.128/g" /var/www/html/config/config.php
      
      # Ou editar manualmente
      docker compose exec nextcloud vi /var/www/html/config/config.php
      # Mudar 'dbhost' => 'postgres' ou IP local do S2
      
      # Reiniciar Nextcloud
      docker compose restart nextcloud

Cenário de Teste 2: Failover Com Certificado Wildcard
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Configuração do Ambiente:**

- Certificados SSL wildcard (``*.librecode.coop``)
- Projetos replicados: ``nginx-proxy``, ``nextcloud-docker``, ``postgres``
- Dados replicados: Nextcloud, PostgreSQL, **certificados SSL**
- **S1 Ativo**: nginx-proxy, nextcloud, postgres (primário)
- **S2 Ativo**: nginx-proxy, nextcloud (standby), postgres (réplica)

**Vantagens do Wildcard:**

- Mesmo certificado funciona em ambos os servidores
- nginx-proxy pode estar ativo em ambos os datacenters
- Failover mais rápido (sem necessidade de iniciar nginx-proxy)
- Menos pontos de falha

**Procedimento de Teste:**

1. **Simular Falha do S1** (Tempo: 1 min)

   .. code-block:: bash
   
      # No S1
      docker compose down

2. **Atualizar DNS** (Tempo: 2-5 min)

   .. code-block:: bash
   
      # Atualizar entrada A para IP do S2
      # lugar.librecode.coop: 189.39.218.225 → 187.57.255.128

3. **Promover PostgreSQL no S2** (Tempo: 30 seg - 1 min)

   .. code-block:: bash
   
      docker compose exec -u postgres postgres pg_ctl promote
      docker compose exec postgres psql -U postgres -c "SELECT pg_is_in_recovery();"

4. **Validar Serviços** (Tempo: 1-2 min)

   .. code-block:: bash
   
      # nginx-proxy já está ativo
      docker compose ps
      
      # Testar acesso
      curl -I https://lugar.librecode.coop
      
      # Verificar certificado SSL
      openssl s_client -connect lugar.librecode.coop:443 -servername lugar.librecode.coop < /dev/null 2>/dev/null | openssl x509 -noout -dates

**Tempo Total Estimado: 5-10 minutos**

**Vantagens Observadas:**

- Failover 50% mais rápido que Cenário 1
- Menos passos manuais
- Certificado válido imediatamente
- nginx-proxy já operacional

**Problemas Específicos:**

- Nenhum problema adicional observado
- Todos os problemas do Cenário 1 ainda aplicáveis (GlusterFS, banco de dados)

Recomendações Pós-Teste
^^^^^^^^^^^^^^^^^^^^^^^^^^

1. **Documentar Tempos Reais**

   .. code-block:: text
   
      Teste realizado em: ___/___/2025
      Horário de início: __:__
      
      Etapa 1 (Desligar S1): __ min
      Etapa 2 (Atualizar DNS): __ min
      Etapa 3 (Promover Postgres): __ min
      Etapa 4 (Iniciar nginx-proxy): __ min (se aplicável)
      Etapa 5 (Validar Nextcloud): __ min
      
      Tempo total: __ min
      Problemas encontrados: ________________

2. **Automatizar Failover**

   Considerar implementar:
   
   - Script de failover automático
   - Monitoramento com ações automáticas
   - Atualização de DNS via API
   - Migrar para Patroni para failover totalmente automático

3. **Melhorias de Infraestrutura**

   - Usar certificados wildcard (se possível)
   - Sincronizar nginx-proxy via GlusterFS
   - Implementar health checks automáticos
   - Configurar alertas para todos os componentes

Procedimento de Teste Trimestral
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   #!/bin/bash
   # dr-test.sh - Executar em ambiente de teste
   
   echo "=== Teste de Disaster Recovery - $(date) ==="
   
   # 1. Simular falha do datacenter primário
   echo "1. Desligando DC1..."
   docker compose -f docker-compose-dc1.yml down
   
   # 2. Promover DC2
   echo "2. Promovendo DC2..."
   docker compose exec patroni1-dc2 patronictl failover cluster --force
   
   # 3. Validar serviços
   echo "3. Validando serviços..."
   curl -f https://nextcloud-test.empresa.com/status.php || exit 1
   
   # 4. Testar operações
   echo "4. Testando upload/download..."
   # Simular uploads e downloads
   
   # 5. Restaurar DC1
   echo "5. Restaurando DC1..."
   docker compose -f docker-compose-dc1.yml up -d
   
   # 6. Switchover de volta
   echo "6. Switchover de volta para DC1..."
   docker compose exec patroni1-dc1 patronictl switchover cluster \
     --leader patroni1-dc2 --candidate patroni1-dc1 --force
   
   echo "=== Teste concluído com sucesso ==="

Documentação do Teste
~~~~~~~~~~~~~~~~~~~~~

Após cada teste, documentar:

- Data e hora do teste
- Cenário testado
- Tempo de recuperação (RTO real)
- Perda de dados (RPO real)
- Problemas encontrados
- Ações corretivas necessárias

Monitoramento e Alertas
------------------------

Métricas Críticas para DR
~~~~~~~~~~~~~~~~~~~~~~~~~~

1. **Lag de Replicação PostgreSQL**

   .. code-block:: bash
   
      # Alerta se lag > 100MB
      docker compose exec patroni1 psql -U postgres -t -c "
        SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)
        FROM pg_stat_replication;"

2. **Status de Sincronização GlusterFS**

   .. code-block:: bash
   
      # Alerta se heal pending > 0
      gluster volume heal volume-nextcloud info | grep "Number of entries"

3. **Idade do Último Backup**

   .. code-block:: bash
   
      # Alerta se último backup > 24h
      LAST_BACKUP=$(aws s3 ls s3://backup-nextcloud/full/ --recursive | tail -1 | awk '{print $1" "$2}')
      AGE=$(( ($(date +%s) - $(date -d "$LAST_BACKUP" +%s)) / 3600 ))
      if [ $AGE -gt 24 ]; then
        echo "ALERTA: Último backup tem $AGE horas"
      fi

4. **Disponibilidade de Datacenters**

   .. code-block:: bash
   
      # Ping entre datacenters
      ping -c 3 dc2-server.empresa.com || echo "ALERTA: DC2 inacessível"

Script de Monitoramento Integrado
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   #!/bin/bash
   # monitor-dr.sh
   
   WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK"
   
   # Função para enviar alerta
   alert() {
     local message="$1"
     echo "ALERTA: $message"
     curl -X POST -H 'Content-type: application/json' \
       --data "{\"text\":\"🚨 DR Alert: $message\"}" \
       $WEBHOOK_URL
   }
   
   # Verificar lag de replicação
   LAG=$(docker compose exec patroni1 psql -U postgres -t -c \
     "SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) 
      FROM pg_stat_replication LIMIT 1;" | tr -d ' ')
   
   if [ "$LAG" -gt 104857600 ]; then  # 100MB
     alert "Replicação atrasada: $(($LAG/1024/1024))MB"
   fi
   
   # Verificar último backup
   LAST_BACKUP_AGE=$(find /backup/full -name "*.tar.gz" -type f -printf '%T@ %p\n' | 
     sort -n | tail -1 | awk '{print (systime() - $1)/3600}')
   
   if [ "$LAST_BACKUP_AGE" -gt 24 ]; then
     alert "Último backup tem ${LAST_BACKUP_AGE}h"
   fi
   
   # Verificar conectividade DC2
   if ! ping -c 1 dc2-server.empresa.com > /dev/null 2>&1; then
     alert "Datacenter secundário inacessível!"
   fi

Configuração Zabbix
~~~~~~~~~~~~~~~~~~~

Templates para monitorar:

- PostgreSQL replication lag
- GlusterFS health status
- Backup job completion
- Network connectivity between DCs
- Disk space on backup volumes

Contatos de Emergência
-----------------------

.. code-block:: text

   ┌─────────────────────────────────────────┐
   │      CONTATOS DE EMERGÊNCIA DR          │
   ├─────────────────────────────────────────┤
   │ Coordenador DR: João Silva              │
   │ Tel: +55 11 9999-9999                   │
   │ Email: joao.silva@empresa.com           │
   ├─────────────────────────────────────────┤
   │ DBA PostgreSQL: Maria Santos            │
   │ Tel: +55 11 8888-8888                   │
   │ Email: maria.santos@empresa.com         │
   ├─────────────────────────────────────────┤
   │ Infra/Storage: Pedro Costa              │
   │ Tel: +55 11 7777-7777                   │
   │ Email: pedro.costa@empresa.com          │
   ├─────────────────────────────────────────┤
   │ Suporte 24/7: +55 11 3000-0000          │
   │ Email: suporte@empresa.com              │
   └─────────────────────────────────────────┘

Checklist de Ações Pós-Desastre
--------------------------------

Após recuperação do desastre, executar:

.. code-block:: text

   ☐ Validar todos os serviços operacionais
   ☐ Verificar integridade dos dados
   ☐ Testar funcionalidades críticas
   ☐ Notificar usuários sobre retorno ao normal
   ☐ Documentar o incidente
   ☐ Realizar análise post-mortem
   ☐ Identificar melhorias no plano DR
   ☐ Atualizar procedimentos conforme aprendizado
   ☐ Agendar próximo teste de DR
   ☐ Revisar backups e logs do incidente

Documentação do Incidente
~~~~~~~~~~~~~~~~~~~~~~~~~~

Template de relatório:

.. code-block:: text

   RELATÓRIO DE INCIDENTE - DISASTER RECOVERY
   ==========================================
   
   Data/Hora do Incidente: _______________
   Detectado por: _______________________
   Tipo de Desastre: ____________________
   
   TIMELINE:
   - HH:MM - Incidente detectado
   - HH:MM - Equipe DR acionada
   - HH:MM - Início da recuperação
   - HH:MM - Serviços restaurados
   - HH:MM - Validação completa
   
   RTO Planejado: ___ horas
   RTO Real: ___ horas
   
   RPO Planejado: ___ minutos
   RPO Real: ___ minutos
   
   Perda de Dados: [ ] Sim [ ] Não
   Se sim, descrever: ___________________
   
   Causa Raiz: __________________________
   
   Ações Corretivas:
   1. _________________________________
   2. _________________________________
   3. _________________________________
   
   Lições Aprendidas:
   1. _________________________________
   2. _________________________________
   3. _________________________________
   
   Assinaturas:
   Coordenador DR: ____________ Data: ___
   Gerente TI: ________________ Data: ___

Referências
-----------

- `PostgreSQL High Availability <https://www.postgresql.org/docs/current/high-availability.html>`_
- `Patroni Standby Cluster <https://patroni.readthedocs.io/en/latest/standby_cluster.html>`_
- `GlusterFS Administration Guide <https://docs.gluster.org/en/latest/>`_
- `Nextcloud Backup and Restore <https://docs.nextcloud.com/server/latest/admin_manual/maintenance/backup.html>`_
- `Disaster Recovery Best Practices <https://www.nist.gov/publications/guide-test-training-and-exercise-programs-it-plans-and-capabilities>`_
