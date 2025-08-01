=======================================
Configurações no sistema hospedeiro
=======================================

É necessário ter instalado o **Docker Engine** e `Docker Compose <https://docs.docker.com/compose/install/standalone/>`_.


Segurança do servidor
=====================

Acesso remoto
-------------

O primeiro passo é criar uma chave ``ssh`` para acessar o servidor.

Vamos utilizar o padrão **ed25519**, o qual é um padrão de chaves públicas mais recente.

Para criar uma nova chave, no terminal, digite o seguinte:

.. note::
   Substitua ``user`` por seu nome ou identificador único.

A chave será criada no seu diretório ``home`` dentro da pasta ``.ssh``.

.. code-block:: bash

   ssh-keygen -t ed25519 -q -f ~/.ssh/<user>_ed25519

Copie a sua chave para o servidor, para possibilitar fazer login sem a utilização de senhas:

.. code-block:: bash

   ssh-copy-id -i ~/.ssh/<user>_ed25519 usuário-no-servidor@ip-ou-nome-do-servidor

No servidor, verifique se a chave está configurada corretamente:

.. code-block:: bash

   cat ~/.ssh/authorized_keys

A chave deve ser a mesma que foi criada com o comando ``ssh-keygen``:

.. important::
   **Nota: apenas copie a chave pública para o servidor, a qual tem a extensão .pub**

No seu computador, verifique se a chave corresponde a que está no servidor:

.. code-block:: bash

   cat ~/.ssh/<user>_ed25519

Teste se está funcionando (não deve solicitar senha ao fazer login):

.. code-block:: bash

   ssh -i ~/.ssh/<user>_ed25519 usuário-no-servidor@ip-ou-nome-do-servidor

O próximo passo é fazer alteração no serviço do ``ssh`` no servidor para que aceite apenas login utilizando chave pública.

Ou seja, apenas quem tiver a sua chave pública no servidor poderá realizar o login no mesmo.

Serviço sshd no servidor
-------------------------

No arquivo ``/etc/ssh/sshd_config``, modifique as seguintes variáveis (ou descomente) para que tenham os seguintes valores:

.. code-block:: bash

   PermitRootLogin prohibit-password
   PubkeyAuthentication yes
   PasswordAuthentication no

Dessa maneira, será possível acessar com o usuário ``root`` mas sem a utilização de senha, apenas com autenticação de chaves.

Considere alterar a porta que o serviço é executado e o número máximo de tentativas permitidas, nas variáveis `Port` e `MaxAuthTries` respectivamente:

.. code-block:: bash

   Port 22003 # Porta aleatória
   MaxAuthTries 3 # Número máximo de tentativas

Será necessário reiniciar o serviço para que as modificações tenham efeito.

Em sistemas debian like poderá ser realizado com o comando ``systemctl reload sshd``

Fail2Ban
--------

De maneira a bloquear tentativas de acessos não autorizados ao servidor, ou ataques de negação de serviço, é possível configurar o **Fail2Ban** para bloquear esses endereços.


Próximo passo
-------------

Realizar as configurações necessárias no registro de nomes (DNS).
