GlusterFS - Sistema de Arquivos Distribuído
============================================

GlusterFS é um sistema de arquivos distribuído escalável e de código aberto que agrega armazenamento de discos em uma única interface de acesso. É ideal para ambientes Nextcloud distribuídos que necessitam de alta disponibilidade e sincronização de dados entre múltiplos datacenters.

Visão Geral
-----------

Características Principais
~~~~~~~~~~~~~~~~~~~~~~~~~~~

- **Descentralizado**: Não possui MDS (Metadata Server) centralizado
- **Escalabilidade Horizontal**: Adicione capacidade adicionando mais servidores
- **Alta Disponibilidade**: Réplicas automáticas entre bricks
- **POSIX-Compliant**: Pode ser montado como sistema de arquivos nativo
- **Geo-Replicação**: Suporte para replicação entre datacenters geograficamente distribuídos
- **Self-Healing**: Recuperação automática de dados corrompidos

Comparação com Outras Soluções
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

+-------------------------+----------------+----------------+---------------+
|                         | Ceph           | GlusterFS      | HDFS          |
+=========================+================+================+===============+
| Arquitetura             | Distribuída    | Descentralizada| Centralizada  |
+-------------------------+----------------+----------------+---------------+
| Gerenciamento           | Múltiplos MDSs | Sem MDS        | Um MDS        |
| de Metadados            |                |                |               |
+-------------------------+----------------+----------------+---------------+
| Método de               | Baseado em     | Baseado em     | Baseado em    |
| Armazenamento           | Objeto         | Arquivo        | Bloco         |
| Subjacente              |                |                |               |
+-------------------------+----------------+----------------+---------------+
| Modelo de               | Horizontal     | Horizontal     | Horizontal    |
| Escalabilidade          |                |                |               |
+-------------------------+----------------+----------------+---------------+
| Caso de Uso             | Armazenamento  | Sistema de     | Armazenamento |
| Principal               | Unificado      | Arquivos       | de Big Data   |
+-------------------------+----------------+----------------+---------------+
| Interface de            | Arquivo, Bloco,| Arquivo        | Arquivo       |
| Armazenamento           | Objeto         |                |               |
+-------------------------+----------------+----------------+---------------+

Conceitos Fundamentais
----------------------

Arquitetura
~~~~~~~~~~~

**Brick (Tijolo)**
  Unidade básica de armazenamento. Cada brick é um diretório exportado em um servidor.

**Volume**
  Coleção lógica de bricks. É o ponto de montagem usado pelos clientes.

**Trusted Storage Pool (TSP)**
  Conjunto de servidores de armazenamento que formam o cluster GlusterFS.

**Client**
  Qualquer máquina que monta um volume GlusterFS.

Tipos de Volumes
~~~~~~~~~~~~~~~~

1. **Distributed (Distribuído)**
   
   - Arquivos distribuídos entre bricks
   - Sem redundância
   - Máximo de espaço disponível
   
   .. code-block:: bash
   
      gluster volume create dist-vol server{1..4}:/data/brick1/dist force

2. **Replicated (Replicado)**
   
   - Cópias idênticas em múltiplos bricks
   - Alta disponibilidade
   - Recomendado para Nextcloud
   
   .. code-block:: bash
   
      gluster volume create repl-vol replica 3 server{1..3}:/data/brick1/repl force

3. **Distributed-Replicated (Híbrido)**
   
   - Combina distribuição e replicação
   - Escalabilidade + redundância
   
   .. code-block:: bash
   
      gluster volume create dist-repl replica 2 server{1..4}:/data/brick1/dr force

4. **Dispersed (Erasure Coding)**
   
   - Similar ao RAID 5/6
   - Melhor eficiência de espaço que replicado
   
   .. code-block:: bash
   
      gluster volume create disp-vol disperse 3 server{1..3}:/data/brick1/disp force

Instalação e Configuração
--------------------------

Pré-requisitos
~~~~~~~~~~~~~~

- Servidores com discos dedicados para bricks (recomendado)
- Resolução de nomes DNS configurada
- Firewall configurado para permitir:
  
  - TCP: 111 (RPC)
  - TCP: 24007 (GlusterD daemon)
  - TCP: 24008 (Management)
  - TCP: 24009+ (Um para cada brick)

.. warning::
   Cada volume montado usa uma porta TCP adicional. Se você tem 3 volumes, precisa liberar portas 24009, 24010 e 24011.

Configuração de DNS
~~~~~~~~~~~~~~~~~~~

Adicione entradas tipo A para cada servidor:

.. code-block:: text

   server1.librecode.coop - 189.39.218.225
   server2.librecode.coop - 187.57.255.128
   server3.librecode.coop - 177.100.50.20

Configuração de /etc/hosts
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Em **todos** os servidores, adicione:

.. code-block:: bash

   # /etc/hosts
   
   # GlusterFS Cluster
   192.168.1.10  server1.librecode.coop server1
   192.168.1.11  server2.librecode.coop server2
   192.168.1.12  server3.librecode.coop server3

.. note::
   Use IPs da rede privada/interna para comunicação entre bricks.

Instalação do GlusterFS
~~~~~~~~~~~~~~~~~~~~~~~~

Em todos os servidores:

.. code-block:: bash

   # Debian/Ubuntu
   apt update
   apt install glusterfs-server -y
   
   # Iniciar e habilitar serviço
   systemctl start glusterd
   systemctl enable glusterd
   systemctl status glusterd

.. code-block:: bash

   # CentOS/RHEL
   yum install centos-release-gluster
   yum install glusterfs-server -y
   
   systemctl start glusterd
   systemctl enable glusterd

Preparar Diretórios de Brick
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Em todos os servidores:

.. code-block:: bash

   # Criar diretório de brick
   mkdir -p /data/brick1/gv0
   
   # Ajustar permissões
   chmod 0711 /data/brick1
   chown root:root /data/brick1

.. tip::
   **Best Practice**: Use um disco/partição separado para bricks:
   
   .. code-block:: bash
   
      # Formatar disco dedicado
      mkfs.xfs -i size=512 /dev/sdb1
      
      # Montar
      echo "/dev/sdb1 /data/brick1 xfs defaults 0 0" >> /etc/fstab
      mount -a
      
      # Criar subdiretório
      mkdir -p /data/brick1/gv0

Criar o Trusted Storage Pool
-----------------------------

No **servidor 1** (inicializador do cluster):

.. code-block:: bash

   # Adicionar server2 ao pool
   gluster peer probe server2.librecode.coop
   
   # Adicionar server3 ao pool
   gluster peer probe server3.librecode.coop
   
   # Verificar status do pool
   gluster peer status

Saída esperada:

.. code-block:: text

   Number of Peers: 2
   
   Hostname: server2.librecode.coop
   Uuid: fa9c3f27-0b60-4718-93bd-b54b79e84e66
   State: Peer in Cluster (Connected)
   
   Hostname: server3.librecode.coop
   Uuid: 3418bdcc-8f9d-4082-993b-121656fcea14
   State: Peer in Cluster (Connected)

.. note::
   Não é necessário executar ``gluster peer probe`` de todos os servidores para todos. Fazendo do server1 para server2 e server3 já é suficiente.

Criar Volume Replicado
~~~~~~~~~~~~~~~~~~~~~~~

No **servidor 1**:

.. code-block:: bash

   # Criar volume replicado entre 3 servidores
   gluster volume create gv0 replica 3 \
     server1.librecode.coop:/data/brick1/gv0 \
     server2.librecode.coop:/data/brick1/gv0 \
     server3.librecode.coop:/data/brick1/gv0 \
     force

Saída esperada:

.. code-block:: text

   volume create: gv0: success: please start the volume to access data

Iniciar Volume:

.. code-block:: bash

   # Iniciar volume
   gluster volume start gv0
   
   # Verificar informações
   gluster volume info gv0

Saída esperada:

.. code-block:: text

   Volume Name: gv0
   Type: Replicate
   Volume ID: 12345678-1234-1234-1234-123456789012
   Status: Started
   Snapshot Count: 0
   Number of Bricks: 1 x 3 = 3
   Transport-type: tcp
   Bricks:
   Brick1: server1.librecode.coop:/data/brick1/gv0
   Brick2: server2.librecode.coop:/data/brick1/gv0
   Brick3: server3.librecode.coop:/data/brick1/gv0

Montar Volume nos Clientes
---------------------------

Instalação do Cliente GlusterFS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Debian/Ubuntu
   apt install glusterfs-client -y
   
   # CentOS/RHEL
   yum install glusterfs-client -y

Montagem Manual
~~~~~~~~~~~~~~~

.. code-block:: bash

   # Criar ponto de montagem
   mkdir -p /mnt/nextcloud-data
   
   # Montar volume
   mount.glusterfs server1.librecode.coop:/gv0 /mnt/nextcloud-data
   
   # Verificar montagem
   df -h | grep gv0
   ls -la /mnt/nextcloud-data

.. note::
   Você pode usar qualquer servidor do cluster como ponto de conexão. O GlusterFS se encarrega de distribuir a carga.

Montagem Automática via /etc/fstab
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Adicionar ao /etc/fstab
   echo "server1.librecode.coop:/gv0 /mnt/nextcloud-data glusterfs defaults,_netdev 0 0" >> /etc/fstab
   
   # Testar
   mount -a
   df -h | grep gv0

Parâmetros de montagem recomendados:

.. code-block:: text

   server1:/gv0 /mnt/data glusterfs defaults,_netdev,backup-volfile-servers=server2:server3 0 0

- ``defaults``: Opções padrão
- ``_netdev``: Aguarda rede estar disponível antes de montar
- ``backup-volfile-servers``: Servidores alternativos se server1 estiver offline

Teste de Replicação
~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Criar 100 arquivos de teste
   for i in $(seq -w 1 100); do 
     cp /var/log/dpkg.log /mnt/nextcloud-data/test-file-$i
   done
   
   # Verificar em todos os servidores
   # No server1
   ls -l /data/brick1/gv0/ | wc -l
   
   # No server2
   ls -l /data/brick1/gv0/ | wc -l
   
   # No server3
   ls -l /data/brick1/gv0/ | wc -l
   
   # Todos devem mostrar 100 arquivos

Segurança
---------

Restrição de Acesso por IP
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Permitir apenas IPs específicos
   gluster volume set gv0 auth.allow "192.168.1.10,192.168.1.11,192.168.1.12"
   
   # Permitir subredes
   gluster volume set gv0 auth.allow "192.168.10.*,192.168.11.*"

Restrição por Diretório
~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Sintaxe: /caminho(IPs_permitidos)
   gluster volume set gv0 auth.allow \
     "/(192.168.10.*|192.168.11.*),\
      /nextcloud(192.168.1.*),\
      /backup(192.168.2.*)"

Explicação:
- ``/``: Raiz acessível por 192.168.10.* e 192.168.11.*
- ``/nextcloud``: Acessível por 192.168.1.*
- ``/backup``: Acessível por 192.168.2.*

SSL/TLS para Comunicação
~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Habilitar SSL no volume
   gluster volume set gv0 client.ssl on
   gluster volume set gv0 server.ssl on
   
   # Gerar certificados
   cd /etc/ssl
   openssl genrsa -out glusterfs.key 2048
   openssl req -new -x509 -key glusterfs.key -out glusterfs.pem -days 365
   
   # Copiar para todos os servidores
   scp glusterfs.* server2:/etc/ssl/
   scp glusterfs.* server3:/etc/ssl/

Monitoramento
-------------

Status do Volume
~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Status básico
   gluster volume status gv0
   
   # Status detalhado
   gluster volume status gv0 detail
   
   # Clientes conectados
   gluster volume status gv0 clients

Saída do status detalhado:

.. code-block:: text

   Brick                : server1:/data/brick1/gv0
   TCP Port             : 49152
   RDMA Port            : 0
   Online               : Y
   Pid                  : 12345
   File System          : xfs
   Device               : /dev/sdb1
   Mount Options        : rw,relatime,attr2
   Disk Space Free      : 450.0GB
   Total Disk Space     : 500.0GB
   Inode Size           : 512
   Inode Count          : 262144000
   Free Inodes          : 262143950

Arquivos Mais Acessados
~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Top 30 arquivos mais lidos
   gluster volume top gv0 read list-cnt 30
   
   # Top 30 arquivos mais escritos
   gluster volume top gv0 write list-cnt 30
   
   # Top 30 diretórios mais abertos
   gluster volume top gv0 opendir list-cnt 30
   
   # Top 30 diretórios mais lidos
   gluster volume top gv0 readdir list-cnt 30

Performance do Volume
~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Profile do volume (iniciar)
   gluster volume profile gv0 start
   
   # Ver informações de profile
   gluster volume profile gv0 info
   
   # Parar profile
   gluster volume profile gv0 stop

Logs
~~~~

.. code-block:: bash

   # Logs do glusterd
   tail -f /var/log/glusterfs/glusterd.log
   
   # Logs de um brick específico
   tail -f /var/log/glusterfs/bricks/data-brick1-gv0.log
   
   # Logs do cliente (mount point)
   tail -f /var/log/glusterfs/mnt-nextcloud-data.log

Integração com Nextcloud
-------------------------

Configuração Recomendada
~~~~~~~~~~~~~~~~~~~~~~~~~

Para melhor performance com Nextcloud:

1. **Não sincronize o código** - Apenas sincronize o diretório de dados dos usuários
2. **Use disco dedicado** - SSDs para melhor IOPS
3. **Ajuste o datadirectory** no Nextcloud

Migrar Data Directory
~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # 1. Colocar Nextcloud em manutenção
   docker compose exec nextcloud php occ maintenance:mode --on
   
   # 2. Alterar datadirectory no config.php
   docker compose exec nextcloud php occ config:system:set datadirectory \
     --value='/mnt/nextcloud-data'
   
   # 3. Mover dados mantendo permissões
   rsync -avz --progress /var/www/html/data/ /mnt/nextcloud-data/
   
   # 4. Ajustar permissões
   docker compose exec nextcloud chown -R www-data:www-data /mnt/nextcloud-data
   
   # 5. Verificar
   docker compose exec nextcloud ls -la /mnt/nextcloud-data
   
   # 6. Desligar manutenção
   docker compose exec nextcloud php occ maintenance:mode --off
   
   # 7. Escanear arquivos
   docker compose exec nextcloud php occ files:scan --all

Docker Compose com GlusterFS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: yaml

   version: '3.8'
   
   services:
     nextcloud:
       image: nextcloud:latest
       container_name: nextcloud
       volumes:
         - /mnt/nextcloud-data:/var/www/html/data:rw
         - nextcloud-config:/var/www/html/config
         - nextcloud-apps:/var/www/html/custom_apps
       environment:
         - NEXTCLOUD_DATA_DIR=/var/www/html/data
   
   volumes:
     nextcloud-config:
     nextcloud-apps:

Otimização de Performance
~~~~~~~~~~~~~~~~~~~~~~~~~~

Parâmetros de volume recomendados:

.. code-block:: bash

   # Performance
   gluster volume set gv0 performance.cache-size 1GB
   gluster volume set gv0 performance.io-thread-count 32
   gluster volume set gv0 performance.read-ahead-page-count 16
   gluster volume set gv0 performance.write-behind-window-size 4MB
   
   # Network
   gluster volume set gv0 network.ping-timeout 10
   gluster volume set gv0 network.remote-dio enable
   
   # Client
   gluster volume set gv0 client.event-threads 4
   
   # Metadata cache
   gluster volume set gv0 performance.md-cache-timeout 600
   gluster volume set gv0 performance.stat-prefetch on

Operações Avançadas
-------------------

Adicionar Brick ao Volume
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Adicionar novo servidor ao pool
   gluster peer probe server4.librecode.coop
   
   # Expandir volume (deve manter o fator de réplica)
   gluster volume add-brick gv0 replica 4 \
     server4.librecode.coop:/data/brick1/gv0
   
   # Rebalancear dados
   gluster volume rebalance gv0 start
   
   # Verificar progresso
   gluster volume rebalance gv0 status

Remover Brick do Volume
~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Reduzir réplicas (força replicação dos dados restantes)
   gluster volume remove-brick gv0 replica 2 \
     server3.librecode.coop:/data/brick1/gv0 start
   
   # Verificar progresso
   gluster volume remove-brick gv0 replica 2 \
     server3.librecode.coop:/data/brick1/gv0 status
   
   # Confirmar remoção
   gluster volume remove-brick gv0 replica 2 \
     server3.librecode.coop:/data/brick1/gv0 commit

Replace Brick (Substituir Brick Com Falha)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Substituir brick com falha
   gluster volume replace-brick gv0 \
     server2.librecode.coop:/data/brick1/gv0 \
     server2.librecode.coop:/data/brick2/gv0 \
     commit force

Snapshots
~~~~~~~~~

.. code-block:: bash

   # Criar snapshot
   gluster snapshot create snap1 gv0
   
   # Listar snapshots
   gluster snapshot list
   
   # Informações do snapshot
   gluster snapshot info snap1
   
   # Ativar snapshot (monta como volume read-only)
   gluster snapshot activate snap1
   
   # Restaurar de snapshot
   gluster snapshot restore snap1
   
   # Deletar snapshot
   gluster snapshot delete snap1

Geo-Replicação (Multi-Datacenter)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Diferenças entre Replicação e Geo-Replicação:

+--------------------------------+-----------------------------------+
| Replicação de Volumes          | Geo-replicação                    |
+================================+===================================+
| Espelha dados em clusters      | Espelha dados em clusters         |
|                                | distribuídos geograficamente      |
+--------------------------------+-----------------------------------+
| Fornece alta disponibilidade   | Garante backup de dados para      |
|                                | recuperação de desastres          |
+--------------------------------+-----------------------------------+
| Replicação síncrona (cada      | Replicação assíncrona (verifica   |
| operação de arquivo é enviada  | as alterações nos arquivos        |
| entre todos os bricks)         | periodicamente e as sincroniza)   |
+--------------------------------+-----------------------------------+
| Menor latência                 | Maior latência, mas funciona      |
|                                | bem em WANs                       |
+--------------------------------+-----------------------------------+

Configurar Geo-Replicação:

.. code-block:: bash

   # No cluster primário (DC1)
   gluster volume geo-replication gv0 \
     server-dc2.librecode.coop::gv0-replica create push-pem
   
   # Iniciar geo-replicação
   gluster volume geo-replication gv0 \
     server-dc2.librecode.coop::gv0-replica start
   
   # Verificar status
   gluster volume geo-replication gv0 \
     server-dc2.librecode.coop::gv0-replica status

Self-Heal
~~~~~~~~~

.. code-block:: bash

   # Verificar arquivos que precisam de heal
   gluster volume heal gv0 info
   
   # Verificar arquivos em split-brain
   gluster volume heal gv0 info split-brain
   
   # Iniciar heal manual
   gluster volume heal gv0
   
   # Heal completo (full scan)
   gluster volume heal gv0 full

Resolver Split-Brain manualmente:

.. code-block:: bash

   # Escolher brick source para um arquivo específico
   gluster volume heal gv0 split-brain source-brick \
     server1.librecode.coop:/data/brick1/gv0 /path/to/file
   
   # Ou escolher automaticamente o mais recente
   gluster volume heal gv0 split-brain latest-mtime /path/to/file

Solução de Problemas
--------------------

Transport Endpoint Not Connected
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Sintoma:**

.. code-block:: text

   ls: cannot access '/mnt/nextcloud-data': Transport endpoint is not connected

**Causas:**
- Firewall bloqueando comunicação
- glusterd parado em algum servidor
- Volume não iniciado

**Solução:**

.. code-block:: bash

   # 1. Verificar firewall
   ufw status
   iptables -L -n | grep 24007
   
   # 2. Verificar glusterd em todos servidores
   systemctl status glusterd
   
   # 3. Verificar volume
   gluster volume status gv0
   
   # 4. Reiniciar volume se necessário
   gluster volume stop gv0
   gluster volume start gv0
   
   # 5. Remontar
   umount /mnt/nextcloud-data
   mount -a

Permissions Too Liberal
~~~~~~~~~~~~~~~~~~~~~~~

**Sintoma:**

.. code-block:: text

   permissions on mountbroker-root directory are too liberal

**Solução:**

.. code-block:: bash

   # Ajustar permissões do brick
   chmod 0711 /data/brick1
   chown root:root /data/brick1

Exhausted All Volfile Servers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Sintoma:**

.. code-block:: text

   0-glusterfsd-mgmt: Exhausted all volfile servers

**Causa:** Arquivo ``/etc/hosts`` incorreto ou DNS não resolvendo

**Solução:**

.. code-block:: bash

   # Verificar /etc/hosts
   cat /etc/hosts | grep server
   
   # Deve conter:
   # 192.168.1.10 server1.librecode.coop server1
   
   # Testar resolução
   ping server1.librecode.coop
   
   # Corrigir se necessário
   vi /etc/hosts

Peer Not in Cluster
~~~~~~~~~~~~~~~~~~~

**Sintoma:**

.. code-block:: text

   Host server1.librecode.coop is not in 'Peer in Cluster' state

**Solução:**

.. code-block:: bash

   # Verificar UUID conflicts
   cat /var/lib/glusterd/glusterd.info
   
   # Se UUIDs duplicados, remover peer e adicionar novamente
   gluster peer detach server1.librecode.coop force
   gluster peer probe server1.librecode.coop

Porta TCP N/A
~~~~~~~~~~~~~

**Sintoma:** ``TCP Port: N/A`` no ``gluster volume status``

**Causa:** Servidor não consegue se comunicar com outros bricks

**Solução:**

.. code-block:: bash

   # Verificar conectividade
   telnet server2.librecode.coop 24007
   
   # Verificar firewall
   ufw allow from 192.168.1.0/24 to any port 24007:24050 proto tcp
   
   # Reiniciar glusterd
   systemctl restart glusterd

Connection Attempt Failed (Invalid Argument)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Sintoma:**

.. code-block:: text

   [2025-10-31 15:30:19] E [socket.c:3539:socket_connect] 0-gv0-client-1: 
   connection attempt on 127.0.0.1:24007 failed, (Invalid argument)

**Causas:**

1. Tentativa de conectar via localhost quando deveria usar hostname/IP correto
2. IPv6 desabilitado mas GlusterFS tentando usar IPv6
3. Configuração de rede incorreta no volume
4. Arquivo de volume corrompido no cliente

**Solução 1: Corrigir Montagem com Hostname Correto**

.. code-block:: bash

   # Desmontar volume
   umount /mnt/nextcloud-data
   
   # Montar usando hostname/IP real do servidor, não localhost
   mount.glusterfs server1.librecode.coop:/gv0 /mnt/nextcloud-data
   
   # Ou usando IP diretamente
   mount.glusterfs 192.168.1.10:/gv0 /mnt/nextcloud-data
   
   # Atualizar /etc/fstab se necessário
   # DE: localhost:/gv0 /mnt/data glusterfs defaults,_netdev 0 0
   # PARA: server1.librecode.coop:/gv0 /mnt/data glusterfs defaults,_netdev 0 0

**Solução 2: Habilitar IPv6 (se desabilitado)**

Se o IPv6 está desabilitado e GlusterFS tenta usá-lo:

.. code-block:: bash

   # Verificar se IPv6 está desabilitado
   cat /proc/sys/net/ipv6/conf/all/disable_ipv6
   # Se retornar 1, está desabilitado
   
   # Habilitar IPv6 temporariamente
   sysctl -w net.ipv6.conf.all.disable_ipv6=0
   sysctl -w net.ipv6.conf.default.disable_ipv6=0
   
   # Ou forçar GlusterFS a usar apenas IPv4
   # Adicionar ao /etc/fstab:
   # server1:/gv0 /mnt/data glusterfs defaults,_netdev,transport=tcp,xlator-option=transport.address-family=inet 0 0
   
   # Remontar
   umount /mnt/nextcloud-data
   mount -a

**Solução 3: Limpar Cache do Cliente**

.. code-block:: bash

   # Parar processos do GlusterFS cliente
   killall glusterfs
   
   # Limpar arquivos de cache/volume do cliente
   rm -rf /var/lib/glusterd/vols/gv0/run/
   rm -rf /var/run/gluster/
   
   # Remontar volume
   mount.glusterfs server1.librecode.coop:/gv0 /mnt/nextcloud-data

**Solução 4: Verificar e Corrigir /etc/hosts**

.. code-block:: bash

   # Verificar /etc/hosts
   cat /etc/hosts | grep -E "server|localhost"
   
   # Deve ter:
   127.0.0.1       localhost
   192.168.1.10    server1.librecode.coop server1
   192.168.1.11    server2.librecode.coop server2
   
   # NÃO deve ter servidor GlusterFS apontando para 127.0.0.1
   # ERRADO: 127.0.0.1 server1.librecode.coop
   # CORRETO: 192.168.1.10 server1.librecode.coop

**Solução 5: Recriar Volfile do Cliente**

.. code-block:: bash

   # No servidor, obter volfile
   gluster volume info gv0
   
   # No cliente, forçar atualização do volfile
   mount.glusterfs server1.librecode.coop:/gv0 /mnt/nextcloud-data -o fetch-attempts=3
   
   # Verificar logs do cliente
   tail -f /var/log/glusterfs/mnt-nextcloud-data.log

**Validar Solução:**

.. code-block:: bash

   # Verificar montagem
   df -h | grep gv0
   mount | grep gv0
   
   # Testar leitura
   ls -la /mnt/nextcloud-data
   
   # Testar escrita
   touch /mnt/nextcloud-data/test-$(date +%s).txt
   
   # Verificar logs sem erros
   tail -20 /var/log/glusterfs/mnt-nextcloud-data.log | grep -i error

.. warning::
   Nunca use ``localhost`` ou ``127.0.0.1`` para montar volumes GlusterFS em clientes remotos. Sempre use o hostname ou IP real do servidor GlusterFS.

Manutenção e Remoção
--------------------

Remover Servidor do Cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # 1. Reduzir réplica removendo brick do servidor
   gluster volume remove-brick gv0 replica 2 \
     server3.librecode.coop:/data/brick1/gv0 force
   
   # 2. Remover peer
   gluster peer detach server3.librecode.coop force
   
   # 3. Verificar
   gluster peer status

Parar e Deletar Volume
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # 1. Desmontar de todos clientes
   umount /mnt/nextcloud-data
   
   # 2. Parar volume
   gluster volume stop gv0
   
   # 3. Deletar volume
   gluster volume delete gv0
   
   # 4. Limpar dados dos bricks (opcional)
   rm -rf /data/brick1/gv0/*

Comandos Úteis
--------------

Referência Rápida
~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Informações do cluster
   gluster peer status
   gluster pool list
   
   # Informações de volumes
   gluster volume list
   gluster volume info [volume]
   gluster volume status [volume] detail
   
   # Gerenciamento de volumes
   gluster volume start [volume]
   gluster volume stop [volume]
   gluster volume restart [volume]
   
   # Configuração
   gluster volume set [volume] [option] [value]
   gluster volume get [volume] [option]
   gluster volume reset [volume] [option]
   
   # Heal e diagnóstico
   gluster volume heal [volume] info
   gluster volume heal [volume] info split-brain
   gluster volume heal [volume]
   
   # Quotas
   gluster volume quota [volume] enable
   gluster volume quota [volume] limit-usage /dir 10GB

Automação com Ansible
---------------------

Role GlusterFS
~~~~~~~~~~~~~~

Exemplo de playbook:

.. code-block:: yaml

   ---
   - hosts: glusterfs_servers
     become: yes
     roles:
       - glusterfs
     
     vars:
       gluster_volume_name: gv0
       gluster_brick_dir: /data/brick1/gv0
       gluster_mount_point: /mnt/nextcloud-data
       gluster_replica_count: 3

Inventory:

.. code-block:: ini

   [glusterfs_servers]
   server1.exemplo.coop ansible_host=192.168.1.10
   server2.exemplo.coop ansible_host=192.168.1.11
   server3.exemplo.coop ansible_host=192.168.1.12

Executar:

.. code-block:: bash

   ansible-playbook -i inventory.ini glusterfs-playbook.yml

Boas Práticas
-------------

Performance
~~~~~~~~~~~

1. **Use XFS para bricks** - Melhor performance que ext4
2. **Discos dedicados** - Separe OS e dados
3. **SSDs para metadata** - Se possível, use SSD para diretório brick
4. **Network 10Gbps+** - Para grandes volumes de dados
5. **Tune kernel parameters**:

   .. code-block:: bash
   
      # /etc/sysctl.conf
      net.core.rmem_max = 134217728
      net.core.wmem_max = 134217728
      net.ipv4.tcp_rmem = 4096 87380 134217728
      net.ipv4.tcp_wmem = 4096 65536 134217728

Alta Disponibilidade
~~~~~~~~~~~~~~~~~~~~

1. **Mínimo 3 réplicas** - Para quorum adequado
2. **Backup-volfile-servers** - Configure servidores alternativos no mount
3. **Monitoramento ativo** - Use Zabbix, Prometheus
4. **Testes regulares** - Simule falhas mensalmente

Segurança
~~~~~~~~~

1. **Rede privada** - Use VLAN/VPN para comunicação entre bricks
2. **Firewall restritivo** - Libere apenas IPs necessários
3. **SSL/TLS habilitado** - Para ambientes sensíveis
4. **Backups offsite** - Não dependa apenas do GlusterFS

Referências
-----------

- `Documentação Oficial GlusterFS <https://docs.gluster.org/en/latest/>`_
- `GlusterFS Architecture <https://docs.gluster.org/en/latest/Quick-Start-Guide/Architecture/>`_
- `Performance Tuning <https://docs.gluster.org/en/latest/Administrator-Guide/Performance-Testing/>`_
- `Troubleshooting Guide <https://docs.gluster.org/en/latest/Troubleshooting/>`_
