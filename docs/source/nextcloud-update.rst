Atualização do Nextcloud
=======================

* Confira se tem o bash_alias no ambiente para o occ:
  Crie um alias para o comando occ editando o arquivo ``~/.bash_alias`` e colocando o seguinte conteúdo:

  .. code-block:: bash

     alias occ='docker compose exec -u www-data app php occ'

* Confira se tem algum PR pendente para ser aceito e mergeado na branch main do repo https://github.com/LibreCodeCoop/nextcloud-docker/pulls
* Notificar usuários sobre a janela de manutenção
* Encerre as conexões dos usuários com o Onlyoffice gentilmente para não haver perda de dados (lembre-se de reativá-lo após a atualização):

  .. code-block:: bash

     docker exec CONTAINER-ONLYOFFICE documentserver-prepare4shutdown.sh

* Derrube o serviço do Onlyoffice se precisar atualizar o mesmo.
* Confira se o backup do banco de dados e dos arquivos estão sendo feitos em sua ferramenta de backup.
* Faça uma lista dos apps ativos e salve em um arquivo:

  .. code-block:: bash

     occ app:list > app_list.old

  .. warning::
     NUNCA pule versão major ao atualizar o Nextcloud. Sempre vá de uma em uma, exemplo: 24 para 25, 25 para 25 e 26 para 27 repetindo todas as etapas deste passo a passo a cada atualização.

* Alterar a versão do Nextcloud no arquivo ``.docker/app/Dockerfile``. Exemplo: ``nextcloud:31-fpm``
* Fazer build do Nextcloud:

  .. code-block:: bash

     docker compose build --pull

* Se tiver problemas com o cache, pode especificar a versão a ser buildada:

  .. code-block:: bash

     docker compose build --build-arg NEXTCLOUD_VERSION=31-fpm --pull --no-cache

* Entrar no modo de manutenção:

  .. code-block:: bash

     occ maintenance:mode --on

* Confira a versão atual do Nextcloud:

  .. code-block:: bash

     occ --version

* Verificar versão do docker compose, pois pode estar diferente do que está sendo executado
* Levantar o Nextcloud e acompanhar o log até o fim da atualização:

  .. code-block:: bash

     docker compose up -d
     docker compose logs -f --tail=100 app

  Você deverá ver ao final:

  .. code-block::

     Initializing finished

* É preciso rodar mais algumas rotinas de atualização:

  .. code-block:: bash

     occ db:add-missing-columns; \
     occ db:add-missing-indices; \
     occ db:add-missing-primary-keys; \
     occ maintenance:repair --include-expensive

* Atualize os apps:

  .. code-block:: bash

     occ app:update --all

* Faça uma lista dos apps ativos e compare para saber se tem algum app que precisa ativar manualmente na versão mais nova:

  .. code-block:: bash

     occ app:list > app_list.new
     diff app_list.old app_list.new

* Rodar o upgrade (irá tirar do modo de manutenção):

  .. code-block:: bash

     occ upgrade

* Acessar o Nextcloud para ver se está tudo funcionando bem:

  * Vá em ``Administration > Overview`` e veja se tem alguma ação a ser executada, precisa deixar tudo verde.
  * Se tiver o Onlyofice na instância, vá em ``Administration > Onlyoffice`` e clique em nos três botões de salvar para se certificar que está correto.
  * Se tiver o LibreSign na instância, vá em ``Administration > LibreSign`` e confira se está tudo verde.

* Limpar o ambiente:

  .. code-block:: bash

     docker system prune

* Notificar fim da manutenção aos usuários caso necessário

Possíveis problemas após a atualização
-------------------------------------

* ``ERROR: permission denied for schema public``

  Solução:

  .. code-block:: sql

     GRANT CREATE ON SCHEMA public TO <username>

  Ref: https://github.com/nextcloud/server/pull/34645/files

