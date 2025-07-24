Instalação e Configuração do Nextcloud
======================================

Antes da Primeira Execução
--------------------------

Crie um banco de dados e um usuário para a aplicação do Nextcloud. Por exemplo, Postgres:
.. code-block:: bash
    # Acesse a pasta do postgres
    cd ~/projetos/postgres
    # Acesse o Postgres com o comando `psql`
    docker compose exec postgres psql -U postgres
    # Crie um usuário para a aplicação
    CREATE ROLE nextcloud WITH LOGIN;
    # Define uma senha para o usuário
    ALTER USER nextcloud WITH PASSWORD 'UmaSenhaMuitoForteVaiAquiMudaIssoPorFavor';
    # Crie o banco de dados e atribua ao usuário recém criado
    CREATE DATABASE nextcloud WITH OWNER = nextcloud;


Clone o repositório no servidor: 
.. code-block:: bash

    git clone https://github.com/LibreCodeCoop/nextcloud-docker.git


Copie o arquivo ``.env.example`` para ``.env`` e defina os valores:

.. code-block:: bash

    cp .env.example .env

Tabela de Variáveis de Ambiente:

+-----------------------------+---------+---------------------------------------------------------------+
| Variável                    | Serviço | Descrição                                                    |
+=============================+=========+===============================================================+
| ``VIRTUAL_HOST``            | `web`   | Seu domínio                                                  |
+-----------------------------+---------+---------------------------------------------------------------+
| ``LETSENCRYPT_HOST``        | `web`   | Seu domínio                                                  |
+-----------------------------+---------+---------------------------------------------------------------+
| ``LETSENCRYPT_EMAIL``       | `web`   | Email do administrador do sistema                            |
+-----------------------------+---------+---------------------------------------------------------------+
| ``NEXTCLOUD_TRUSTED_DOMAINS``| `app`   | Domínios separados por vírgula. O domínio web é obrigatório. |
+-----------------------------+---------+---------------------------------------------------------------+
| ``POSTGRES_DB``             | `db`    | Nome do banco PostgreSQL (padrão: nextcloud)                 |
+-----------------------------+---------+---------------------------------------------------------------+
| ``POSTGRES_USER``           | `db`    | Usuário do banco PostgreSQL (padrão: nextcloud)              |
+-----------------------------+---------+---------------------------------------------------------------+
| ``POSTGRES_PASSWORD``       | `db`    | Senha do usuário do banco PostgreSQL                         |
+-----------------------------+---------+---------------------------------------------------------------+
| ``POSTGRES_HOST``           | `app`   | Host do servidor PostgreSQL (padrão: postgres)               |
+-----------------------------+---------+---------------------------------------------------------------+
| ``NEXTCLOUD_ADMIN_USER``    | `app`   | Nome de usuário do administrador do Nextcloud                |
+-----------------------------+---------+---------------------------------------------------------------+
| ``NEXTCLOUD_ADMIN_PASSWORD``| `app`   | Senha do administrador do Nextcloud                          |
+-----------------------------+---------+---------------------------------------------------------------+
| ``NEXTCLOUD_ADMIN_EMAIL``   | `app`   | Email do administrador do Nextcloud                          |
+-----------------------------+---------+---------------------------------------------------------------+
| ``SMTP_HOST``               | `app`   | Servidor SMTP para envio de emails                           |
+-----------------------------+---------+---------------------------------------------------------------+
| ``SMTP_SECURE``             | `app`   | Tipo de segurança SMTP (ssl, tls ou vazio)                   |
+-----------------------------+---------+---------------------------------------------------------------+
| ``SMTP_PORT``               | `app`   | Porta do servidor SMTP                                       |
+-----------------------------+---------+---------------------------------------------------------------+
| ``SMTP_AUTHTYPE``           | `app`   | Tipo de autenticação SMTP (LOGIN, PLAIN, NTLM)               |
+-----------------------------+---------+---------------------------------------------------------------+
| ``SMTP_NAME``               | `app`   | Nome de usuário para autenticação SMTP                       |
+-----------------------------+---------+---------------------------------------------------------------+
| ``SMTP_PASSWORD``           | `app`   | Senha para autenticação SMTP                                 |
+-----------------------------+---------+---------------------------------------------------------------+
| ``MAIL_FROM_ADDRESS``       | `app`   | Endereço de email do remetente                               |
+-----------------------------+---------+---------------------------------------------------------------+
| ``MAIL_DOMAIN``             | `app`   | Domínio do email do remetente                                |
+-----------------------------+---------+---------------------------------------------------------------+
| ``TZ``                      | `app`   | Fuso horário (ex: America/Sao_Paulo)                         |
+-----------------------------+---------+---------------------------------------------------------------+

.. important::
   O Let's Encrypt só funciona em servidores onde ``VIRTUAL_HOST`` e ``LETSENCRYPT_HOST`` possuem um domínio público válido registrado em um servidor DNS. Não tente usar em localhost, não funcionará!

Crie as redes necessárias:

.. code-block:: bash

    docker network create reverse-proxy
    docker network create postgres


Subindo os serviços
-------------------

Construa as imagens, e suba os containers:

.. code-block:: bash
    # Construindo imagens
    docker compose build --pull
    # Subindo os containers
    docker compose up -d

    
Após a Configuração
-------------------

Após concluir a configuração, acesse: https://seudominio.com.br/settings/admin/overview

Se for necessário executar qualquer comando occ, execute assim:

.. code-block:: bash
    # Verifique o status da instalação
    docker compose exec -u www-data app ./occ setupchecks

Configuração Personalizada
-------------------------

### Personalizar o conteúdo do docker-compose

Você pode fazer isso usando variáveis de ambiente e criando um arquivo chamado ``docker-compose.override.yml`` para adicionar novos serviços.

### Redis
Adicionando redis para cache de memória. 
1. Crie a rede docker para o `redis`: `docker network create redis`
2. Crie o arquivo `docker-compose.override.yml`
3. Adicione o serviço do redis e adicione a rede do redis aos serviços que vão acessar o mesmo.

.. code-block:: yaml
    networks:
      redis:
          external: true
          name: redis

    services:
      redis:
          image: redis:alpine
          restart: unless-stopped
          volumes:
          - ./volumes/redis/data:/data
          - ./volumes/redis/conf/redis.conf:/usr/local/etc/redis/redis.conf
          networks:
          - redis
      app:
          networks:
          - redis
      cron:
          networks:
          - redis


4. Adicione ao arquivo de configuração do Nextcloud o bloco de configuração do `redis`

.. code-block:: bash
        'memcache.distributed' => '\\OC\\Memcache\\Redis',
        'memcache.locking' => '\\OC\\Memcache\\Redis',
        'redis' => 
        array (
            'host' => 'redis',
        ),



### PHP

1. Crie seu arquivo ``.ini`` na pasta ``volumes/php/``. Exemplo: ``volumes/php/xdebug.ini``
2. Altere o arquivo ``docker-compose.override.yml`` adicionando seu volume:

.. code-block:: yaml

    services:
      app:
        volumes:
          - ./volumes/php/xdebug.ini:/usr/local/etc/php/conf.d/xdebug.ini

### PHP-FPM

Para modificações no PHP-FPM, inclua o seguinte volume no serviço app no arquivo ``docker-compose.override.yml``:

.. code-block:: yaml

    services:
      app:
        volumes:
            - ./volumes/php/pm.ini:/usr/local/etc/php/conf.d/

Crie um arquivo ``./volumes/php/pm.ini`` com o seguinte conteúdo (consulte as referências para ajustes de acordo com sua configuração):

.. code-block:: yaml

    [www]
    pm.max_children = 10
    pm.start_servers = 2
    pm.min_spare_servers = 1
    pm.max_spare_servers = 3

Referências:
- https://docs.nextcloud.com/server/21/admin_manual/installation/server_tuning.html#tune-php-fpm
- https://spot13.com/pmcalculator/

Executando o Nextcloud
----------------------

.. code-block:: bash

    # O serviço postgres é executado separadamente para ser possível reutilizar este serviço para outras aplicações que usam PostgreSQL
    docker compose up -f docker-compose-postgres.yml -d
    docker compose up -d

Usando uma Versão Específica do Nextcloud
-----------------------------------------

Altere o valor de ``NEXTCLOUD_VERSION`` no arquivo ``.env`` e coloque o nome da tag que deseja usar. Verifique as tags disponíveis em: https://hub.docker.com/_/nextcloud/tags



Visualizando Logs
-----------------

Se quiser ver os logs, execute:

.. code-block:: bash

    docker compose logs -f --tail=100

Você verá esta mensagem nos logs entre outras mensagens de atualização:

.. code-block:: log

    app_1      | 2025-06-25T11:10:09.568623133Z Initializing nextcloud 31.0.8.0 ...
    app_1      | 2025-06-25T11:10:09.577733913Z Upgrading nextcloud from 31.0.7.0 ...


Diretório de arquivos
---------------------
O diretório que contém os dados dos usuários se encontra mapeado para o host, no seguinte caminho `./volumes/nextcloud/data`.
Se há a necessidade de mover esses dados, lembre-se de ajustar as permissões posteriormente.
O dono e grupo dos arquivos são o `wwww-data`.


Backup
------
Os seguintes arquivos devem feitos backup com sua ferramenta de preferência. São eles:

.. code-block:: bash
    Configurações: /volumes/nextcloud/config/
    Dados dos usuários: /volumes/nextcloud/data/
    Pasta dos temas: /volumes/nextcloud/themes/
    Compose do projeto: docker-compose.yml
    Secrets: .env