# Microserviço

De maneira simplista, para que uma aplicação possa sobreviver dentro de um cluster de **Kubernetes** ou **Swarm** é preciso que ela seja escalável, mantendo seu comportamento ao incremento ou decremento de suas réplicas.

Isso implica algumas coisas simples de concepção, mas complexas de execução, como quebrar a aplicação em microserviços, persistir o estado fora da aplicação, fraca dependência e etc.

### Uma aplicação não escalável

Imagine uma aplicação simples em PHP que valide acessos através de um formulário de login consultando um banco de dados MySQL e salve seus dados de sessão da maneira padrão, como arquivos no disco local.

Caso essa aplicação precise ser enviada para um outro **nó** do seu cluster o contêiner será destruído e recriado. Isso significa que todo o estado da aplicação foi perdido, e todos os usuários autenticados agora precisam autenticar-se novamente.

O mesmo ocorrerá caso algum problema aconteça com o contêiner da aplicação, ele será destruído e recriado, mesmo que seja na mesma máquina, um novo contêiner trás consigo uma nova camada.

O estado dessa aplicação deverá então ser colocado fora do contâiner, em um sistema de arquivo compartilhado, no banco de dados, a melhor das opções, um servidor de **cache**.

### Exercício

Provisionar toda a infraestrutura para suportar a aplicação que conterá:

 - Namespace
 - Banco de dados MySQL
 - Servidor de cache MemcacheD
 - Aplicação PHP
 - Serviços para MySQL, MemCached e a aplicação
 - Secret com dados de acesso ao banco
 - ConfigMap com arquivo de configuração do PHP

#### Namespace

Criar um **namespace** chamado **login** para englobar todos os objetos do exercício.

##### Persistent Volume

Criar um volume persistente de nome **mysql-pv** do tipo de acesso **RWO** com **1Gi**.

##### Persistent Volume Claim

Criar um PVC pra o volume **mysql-pv** e chamá-lo de **mysql-pvc**.
 
##### MySQL

O banco de dados será o MySQL versão 5.7 provisionado através de um **stateful-set**. Deverá ter um **service** chamado **mysql**.

##### Memcached

O MemcacheD deverá ser provisionado duas vezes, cada um com um **deployment** e um **service**.

##### Secret e ConfigMap

Criar um **secret** chamado **mysql** com os dados de acesso ao banco e um **config-map** chamado **php-ini** com as configurações do php.ini, ambos descritos com mais detalhes abaixo.

##### Aplicação

Provisionar dentro do cluster de Kubernetes um deploy com três réplicas do contêiner **hectorvido/php-app:ha**.
O deployment deverá utilizar um **secret** para popular as variáveis:
 - DB_HOST
 - DB_PORT
 - DB_USER
 - DB_NAME
 - DB_PASS

O deployment também deverá utilizar um **config-map** com o conteúdo a seguir - modificando a sessão **session.save_path** - para ser colocando em */etc/php7/php.ini*:

```
[PHP]
engine = On
short_open_tag = Off
precision = 14
output_buffering = 4096
zlib.output_compression = Off
implicit_flush = Off
unserialize_callback_func =
serialize_precision = -1
disable_functions =
disable_classes =
zend.enable_gc = On
expose_php = On
max_execution_time = 30
max_input_time = 60
memory_limit = 128M
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
display_errors = Off
display_startup_errors = Off
log_errors = On
log_errors_max_len = 1024
ignore_repeated_errors = Off
ignore_repeated_source = Off
report_memleaks = On
html_errors = On
variables_order = "GPCS"
request_order = "GP"
register_argc_argv = Off
auto_globals_jit = On
post_max_size = 8M
auto_prepend_file =
auto_append_file =
default_mimetype = "text/html"
default_charset = "UTF-8"
include_path = ".:/usr/share/php7"
doc_root =
user_dir =
enable_dl = Off
file_uploads = On
upload_max_filesize = 2M
max_file_uploads = 20
allow_url_fopen = On
allow_url_include = Off
default_socket_timeout = 60

[CLI Server]
cli_server.color = On

[MySQLi]
mysqli.max_persistent = -1
mysqli.allow_persistent = On
mysqli.max_links = -1
mysqli.cache_size = 2000
mysqli.default_port = 3306
mysqli.default_socket =
mysqli.default_host =
mysqli.default_user =
mysqli.default_pw =
mysqli.reconnect = Off

[mysqlnd]
mysqlnd.collect_statistics = On
mysqlnd.collect_memory_statistics = Off

[bcmath]
bcmath.scale = 0

[Session]
session.save_handler = memcached
session.save_path = "<servico>:<porta>,<servico>:<porta>"
session.use_strict_mode = 0
session.use_cookies = 1
session.use_only_cookies = 1
session.name = PHPSESSID
session.auto_start = 0
session.cookie_lifetime = 0
session.cookie_path = /
session.cookie_domain =
session.cookie_httponly =
session.serialize_handler = php
session.gc_probability = 1
session.gc_divisor = 1000
session.gc_maxlifetime = 1440
session.referer_check =
session.cache_limiter = nocache
session.cache_expire = 180
session.use_trans_sid = 0
session.sid_length = 26
session.trans_sid_tags = "a=href,area=href,frame=src,form="
session.sid_bits_per_character = 5
```
