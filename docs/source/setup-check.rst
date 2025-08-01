Verificação de segurança
=======================

- Como saber se o sistema foi comprometido?
- É possível utilizar muitos utilitários de linha de comando para fazer uma análise.

Usuários logados
----------------

- O comando ``w`` ou ``who``, mostra quem está logado **atualmente**.
- Já o comando ``last``, informa o histórico de quem acessou o servidor.

Histórico de comandos
---------------------

- No arquivo ``.bash_history`` ficam registrados os comandos executados.
- Podemos verificar os últimos 100 utilizando o comando abaixo:

  .. code-block:: bash

     tail -n 100 ~/.bash_history | more

Arquivos que foram modificados recentemente
------------------------------------------

- Pode-se verificar quais arquivos foram modificados recentemente.
- O comando abaixo mostra quais arquivos foram modificados nos últimos ``5`` dias:

  .. code-block:: bash

     sudo find /home /etc /var -mtime -5

Conexões de rede
----------------

netstat
~~~~~~~

- O comando ``netstat`` permite observarmos as conexões que estão ``chegando`` e ``saindo`` do nosso servidor.
- ``-t`` indica conexões TCP
- ``-u`` indica conexões UDP
- ``-p`` programa que está utilizando a porta
- ``-a`` mostra todas interfaces
- ``-n`` não resolve os nomes de rede

  .. code-block:: bash

     sudo netstat -tupan

lsof
~~~~

- Com o comando ``lsof -u user`` conseguimos ver quais arquivos estão abertos pelo usuário ``user``.
- Da mesma maneira, com o parâmetro ``-i`` podemos observar quais programas estão utilizando endereços de rede.

  .. code-block:: bash

     sudo lsof -i

ssh
~~~

- Verificar tentativas de acesso ``ssh`` ao servidor.
- Com journalctl: ``journalctl -u ssh.service``.
- Adicione o parâmetro ``-f`` para ver apenas as novas entradas.
- O parâmetro ``-n 20`` mostra apenas as últimas 20 entradas.
- ``-r`` mostra em ordem reversa, sendo os mais novos primeiro

rkhunter
--------

- Segundo a documentação:

  .. code-block:: bash

     O Rootkit Hunter verifica os sistemas em busca de rootkits, backdoors, sniffers e exploits conhecidos e desconhecidos.

     Ele verifica:

     Alterações de hash SHA256;
     arquivos comumente criados por rootkits;
     executáveis com permissões de arquivo anômalas;
     strings suspeitas em módulos do kernel;
     arquivos ocultos em diretórios do sistema; e pode, opcionalmente, verificar dentro de arquivos.

- Em distribuições baseadas em Ubuntu, pode-se instalar com: ``apt-get install rkhunter``.
- Uma checagem pode ser executada passando o parâmetro ``-a``: ``rkhunter -a``
- Listar os testes é possível passando o parâmetro ``--list``.

Verificação de performance
=========================

- Antes de fazer a instalação dos serviços no servidor, uma boa prática é conferir se o ambiente entregue está pronto para produção.
- O que podemos testar:

  - [  ] Velocidade de download/upload
  - [  ] Velocidade de escrita em disco - IO
  - [  ] Teste de stress de CPU
  - [  ] Reputação do IP público

Velocidade de download/upload
-----------------------------

.. code-block:: bash

   # Velocidade de Download/Upload
   # Utilizando o repositório oficial
   sudo apt-get install curl -y
   curl -s https://packagecloud.io/install/repositories/ookla/speedtest-cli/script.deb.sh | sudo bash
   sudo apt-get install speedtest -y

   # Com docker - utilizando image de terceiros
   docker run --rm wesbragagt/alpine-speedtest

Velocidade de escrita em disco
------------------------------

- Utilizando o comando ``dd``:

  .. code-block:: bash

     # Velocidade de escrita em disco - Irá criar um arquivo de 10GB contendo '0'
     dd if=/dev/zero of=./testfile bs=1G count=10 oflag=direct status=progress

- Utilizando o comando ``fio``, o qual permite testes mais avançados:

  .. code-block:: bash

     # Teste de escrita aleatória
     fio --name=random-write --ioengine=libaio --rw=randwrite --bs=4k --size=1G --numjobs=1 --runtime=60 --time_based --end_fsync=1

- Segundo a documentação:

  .. code-block:: bash

     Fio gera uma série de threads ou processos que executam um tipo específico de ação de E/S, conforme especificado pelo usuário. 
     fio recebe uma série de parâmetros globais, cada um herdado pela thread, a menos que parâmetros que substituam essa configuração sejam fornecidos. 
     O uso típico de fio é escrever um arquivo de tarefa correspondente à carga de E/S que se deseja simular.

Teste de stress de CPU
----------------------

- Para testes de stress de CPU, pode-se utilizar os seguintes comandos: ``stress``, ``sysbench``.
- Visualize o teste com o comando ``htop``. Execute os comando em segundo plano colocando ``&`` ao final de cada comando.

  .. code-block:: bash

     # Teste por 60 segundos com 4 threads
     stress --cpu 4 --timeout 60s
     # Teste de CPU (primos até 20000)
     sysbench cpu --cpu-max-prime=20000 run

- É possível definir quantas threads serão executadas:
- Para verificar quantas thread tem o seu processador: ``grep processor /proc/cpuinfo | wc -l``

  .. code-block:: bash

     sysbench cpu run --threads=16

- Com ``sysbench`` é possível fazer testes de CPU, Memória, Entrada/Saída, performance de banco de dados.

Reputação do IP público
-----------------------

- Ao adquirir um serviço na nuvem, pode-se verificar a reputação do endereço IP.
- Abaixo estão alguns sites que oferecem maneiras de verificar a reputação bem como
  fazer denúncias caso um IP esteja sendo usado de maneira indevida.
  
  *Para descobrir o seu endereço IP pela linha de comando podes usar ``curl ifconfig.me``*

  - https://www.abuseipdb.com
  - https://www.virustotal.com
  - https://www.spamhaus.org/

Referências
===========

- https://www.kali.org/tools/rkhunter/
- https://fio.readthedocs.io/en/latest/fio_doc.html
- https://wiki.archlinux.org/title/Benchmarking
