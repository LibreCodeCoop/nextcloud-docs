Bem vindo a documentação acerca do Nextcloud em ambientes distribuídos!
===================================

`Nextcloud <https://nextcloud.com>`_ é uma plataforma para hospedagem de arquivos, organização do trabalho, comunicação entre pessoas, além de ser um ecossistema para diversos aplicativos.

É possível executar essa aplicação em pequenas placas (Raspberry Pi) até em grandes centros de dados com alto poder computacional.

Esta documentação tem o objetivo de demonstrar os cenários possíveis para a instalação do Nextcloud:

   - Desde uma instância até um cluster com diversos membros.

.. note::

   Este projeto está em desenvolvimento contínuo.


.. Documentação Principal

==================
Manual de Utilização
==================

.. toctree::
   :maxdepth: 2
   :caption: Introdução
   
   inicio
   setup-check

.. toctree::
   :maxdepth: 2
   :caption: Configuração do Ambiente
   
   hospedeiro
   dns
   proxy-reverso

.. toctree::
   :maxdepth: 2
   :caption: Banco de Dados
   
   banco
   banco-replica

.. toctree::
   :maxdepth: 2
   :caption: Armazenamento
   
   arquivos-sinc
   backup

.. toctree::
   :maxdepth: 2
   :caption: Nextcloud
   
   nextcloud-instalacao
   nextcloud-update