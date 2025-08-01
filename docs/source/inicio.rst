=============================================
Considerações sobre Ambientes Distribuídos
=============================================

Este documento visa trazer considerações sobre ambientes distribuídos e a aplicação Nextcloud.

Nextcloud em ambientes distribuídos
====================================

O Nextcloud é uma aplicação que precisa de, no mínimo:

1. Armazenamento de arquivos
2. Banco de dados
3. Servidor web
4. Linguagem de interpretação (PHP)

Consulte a página de requisitos para se familiarizar. (https://docs.nextcloud.com/server/latest/admin_manual/installation/system_requirements.html)

Tecnologias de Sincronização
----------------------------

Em se tratando de sincronização de dados, após realizar pesquisas, optou-se por realizar testes com **GlusterFS** e **GarageS3**.

**GlusterFS** foi escolhido por sua natureza descentralizada, diferentemente do CEPH que armazena os meta-dados num servidor centralizado. Além disso, ambas tecnologias apresentam maturidade desejável.

**GarageS3** foi escolhido por sua natureza inovadora, de maneira a prover um software capaz de poder ser utilizado em hardware de baixo custo/não confiável e por ter uma natureza distribuída como princípio.

.. note::
   O software MinIO não foi escolhido pois não possui natureza distribuída.

Visto que o Nextcloud tem a opção utilizar *buckets* como armazenamento primário, poderia-se utilizar o provedor de armazenamento em nuvem Wasabi, o qual oferece a possibilidade de replicar os *buckets* em diferentes regiões. Todavia o mesmo provedor não oferece serviços no Brasil.

Banco de Dados Distribuído
---------------------------

Outro aspecto a ser levado em conta é o banco de dados: é necessário fazer a replicação em tempo real.

Existem soluções como **MariaDB Galera Cluster** e **Autobase** que fornecem uma interface gráfica para fazer a instalação e configuração de novos clusters.

Também existe o **Patroni**, o qual é um template para clusters de Postgres, implementando protocolo de consenso (ETCD, por exemplo) banco de dados Postgres.

Optou-se pela replicação nativa, afim de melhor exemplificar.

Coordenação Distribuída
------------------------

Quando fala-se de sistema distribuído, é necessário a utilização de ferramentas como o **etcd**, o qual é um armazenamento chave-valor distribuído e altamente disponível, amplamente utilizado em sistemas distribuídos para armazenamento de configurações compartilhadas, descoberta de serviços e coordenação entre nós.

O etcd atua em conjunto com o Patroni, de maneira a garantir que a aplicação apenas escreva no servidor primário.

A topologia com dois datacenters para fazer o cluster de banco de dados fica da seguinte maneira:

.. image:: patroni-1.png
   :alt: Topologia de dois datacenters, onde um dos dois fica em espera recebendo atualizações do primário, e, torna-se primário em caso de falhas
   :align: center


*Fonte:* https://patroni.readthedocs.io/en/latest/ha_multi_dc.html

Configurações necessárias a nível de DNS
=========================================

Entrada tipo A apontando para o IP da aplicação
Cenário 1: Provedor de DNS sem suporte a certificados *wildcard*
Cenário 2: Provedor de DNS com suporte a certificados *wildcard*

Certificados Wildcard
----------------------

- Caso o provedor não suporte certificados *wildcard*, será necessário, quando a aplicação no Datacenter primário estiver comprometida, adicionar o IP do servidor 2 à entrada tipo A existente.
- Se o provedor DNS suportar certificados *wildcard*, o mesmo deverá ser copiado para os demais servidores após a requisição no servidor 1.

Balanceamento de carga em ambientes distribuídos
=================================================

- Por se tratar de um setup primário/secundário, o balanceamento de carga entre datacenters não será habilitado.
- O que pode ser habilitado é o balanceamento de carga dentro de cada datacenter em questão, a depender dos recursos computacionais disponíveis.



Ferramentas para backup e recuperação em caso de desastres
===========================================================

A ferramenta utilizada para backup (em servidor diferente do cluster) é o **Duplicati**.


Soluções de armazenamento de backup
====================================

O provedor **Wasabi** é o escolhido para armazenamento em objetos. De maneira que o backup será criptografado, não há problema em utilizar provedor externo.
Outra opção é utilizar o MinIO, software livre que permite a criação de um ambiente compatível com o protocolo S3. 

.. warning::
   A única ressalva é a latência do link de internet para restaurar o backup, o que pode demorar.


Ferramentas para monitoramento da infraestrutura
=================================================

A ferramenta **Zabbix** será utilizada afim de monitorar a infraestrutura.

Outras utilitários de linha de comando também podem ser utilizados, tais quais::

   htop - mostra os processos que estão em execução
   iotop - mostra escrita em disco de cada processo
   


Levantamento de ferramentas para infraestrutura como código
============================================================

Quando se é utilizado o virtualizador **Proxmox**, logo, poder-se-ia utilizar a ferramenta **OpenTofu** para instanciar novas máquinas.

Todavia, será utilizado o **Ansible** para automatizar tarefas de maneira geral.

Ferramentas de interconexão utilizando redes definidas por software
====================================================================

Em estudo se será necessário.


Diagramação da arquitetura proposta
====================================

Abaixo está o diagrama da arquitetura proposta:

.. code-block:: mermaid

   graph TB
       subgraph DC1["Datacenter 1"]
           LB1[Load Balancer]
           subgraph SERVER1["Servidor 1"]
               NGINX1["NGINX (SSL Wildcard)"]
               APP1["Aplicação"]
               PG1["PostgreSQL (Primary)"]
               PAT1["Patroni"]
           end
           ETCD1["etcd node"]
           GFS1["GlusterFS Brick"]
       end

       subgraph DC2["Datacenter 2"]
           LB2[Load Balancer]
           subgraph SERVER2["Servidor 2"]
               NGINX2["NGINX (SSL Wildcard)"]
               APP2["Aplicação"]
               PG2["PostgreSQL (Replica)"]
               PAT2["Patroni"]
           end
           ETCD2["etcd node"]
           GFS2["GlusterFS Brick"]
       end

       %% Conexões entre componentes
       LB1 <--> NGINX1
       NGINX1 --> APP1
       APP1 --> PAT1
       PAT1 --> PG1
       
       LB2 <--> NGINX2
       NGINX2 --> APP2
       APP2 --> PAT2
       PAT2 --> PG2

       %% Conexões de replicação e cluster
       PG1 <-.Replicação PostgreSQL.-> PG2
       PAT1 <-.Gestão de Failover.-> PAT2
       
       %% Conexões GlusterFS
       GFS1 <-.Replicação de Volumes.-> GFS2
       APP1 -.-> GFS1
       APP2 -.-> GFS2
       
       %% Conexões etcd
       ETCD1 <-.Cluster ConsensusEleição de Líder.-> ETCD2
       PAT1 -.-> ETCD1 & ETCD2
       PAT2 -.-> ETCD1 & ETCD2
       
       %% Conexões entre datacenters
       LB1 <-.Failover.-> LB2
       
       %% Usuário
       USER((Usuários)) --> LB1
       USER --> LB2
       
       %% Adicionando legenda
       classDef default fill:#f9f9f9,stroke:#333,stroke-width:1px
       classDef lb fill:#f5f5dc,stroke:#333
       classDef nginx fill:#C6E2FF,stroke:#333
       classDef app fill:#D8BFD8,stroke:#333
       classDef db fill:#FFDAB9,stroke:#333
       classDef etcd fill:#98FB98,stroke:#333
       classDef gfs fill:#FFD700,stroke:#333
       classDef patroni fill:#FF6347,stroke:#333
       
       class LB1,LB2 lb
       class NGINX1,NGINX2 nginx
       class APP1,APP2 app
       class PG1,PG2 db
       class ETCD1,ETCD2 etcd
       class GFS1,GFS2 gfs
       class PAT1,PAT2 patroni

Considerações
=============

Limitações com 2 Data Centers
------------------------------

Utilizando 2 data centers para a redundância requer intervenção manual para fazer a ativação do servidor secundário. 
Isso ocorre pois uma das limitações é que com apenas 2 nós não é possível utilizar o etcd para fazer a eleição de um líder.


Ambiente com Alta Disponibilidade em 3 Data Centers
----------------------------------------------------

.. image:: patroni-2.png
   :alt: Ambiente com 3 datacenters
   :align: center

*Fonte:* https://patroni.readthedocs.io/en/latest/ha_multi_dc.html


Próximos passos
---

O próximo passo é realizar configurações no sistema hospedeiro, de maneira a deixá-lo apto para a instalação da aplicação.

.. toctree::
   :numbered:
   :maxdepth: 2
   :hidden:

   hospedeiro
   configuration
   maintenance
   security