**# Configurações no DNS**
========================

- Essas são as instruções de como configurar o seu registro no DNS apropriadamente
- Você deve possuir um registro. Senão o tiver, `registre o seu! <https://registro.br/>`_

**## Entradas a serem inseridas**
-------------------------------

- A quantidade de entradas configuradas irá depender de quantos servidores serão utilizados.
- Caso tenha 3 servidores, configure 3 entradas do tipo A, cada qual apontando ao IP público de cada servidor:
- Por exemplo:

.. image:: https://github.com/user-attachments/assets/0e6665dc-0d9a-4a18-8f7a-18d5b24531d6

**## Como validar se a configuração está funcionando**
----------------------------------------------------

- Utilize o comando a seguir para verificar se o IP do servidor está resolvendo o registro criado:

.. code-block:: bash

   nslookup seudominio.com.br

**## Configurar registros para seus servidores**
----------------------------------------------

- Adicione um subdomínio para facilitar o acesso ao seu servidor.
- Dessa maneira, o acesso ssh poderia ser feito utilizando o nome do servidor:

.. code-block:: bash

   ssh usuario@servidor01.seudominio.com.br

.. image:: https://github.com/user-attachments/assets/fa522441-5b14-431d-a5c5-397dabb01e20

**## Automatizando essa tarefa**
------------------------------

- Para automatizar essa tarefa de criação de registros, o provedor DNS deverá fornecer acesso a uma API.
- Alguns exemplos de provedores que fornecem acesso:
- `Cloudflare <https://developers.cloudflare.com/api/operations/dns-records-for-a-zone-create-dns-record>`_
- `Gandi <https://api.gandi.net/docs/domains/>`_
- `DNSSimple <https://dnsimple.com/api>`_