# Multiplos Contêiners respondendo pela Porta 80 utilizando o NGINX como Proxy reverso - Docker-Compose

## Exemplo
É isso que faremos. Duas aplicações que utilizam docker-compose executados no mesmo servidor, que compartilham o mesmo proxy NGINX, que fara o papael de proxy reverso.

```
          Servidor:
         |-------------------------------------------|
         |                                           |
         |           |--- App 1 (app1.example.com)   |
Cliente --- NGINX ---|                               |
         |           |--- App 2 (app2.example.com)   |
         |                                           |
         |-------------------------------------------|
```

## Problema
Você não pode incluir um proxy NGINX em cada aplicativo, porque então você teria vários proxies tentando escutar na porta 80/443. Você só pode ter um.

Infelizmente, o docker-compose não possui nenhum suporte embutido para serviços "singleton" que são compartilhados entre projetos. Ou seja não há como dizer: "Crie este serviço somente se ele já não existir".

## Solução
Minha solução é dividir o exemplo acima em 3 projetos e compartilhar a mesma rede em todos os projetos.

## Detalhando a Solução
### proxy_reverso
Neste diretório está configurado o Proxy que escutará a porta 80 e recebera todas as requisições, encaminhando cada requisição para o contêiner correto.

#### Entendendo a estrutura:
Neste diretório temos um arquivo `docker-compose.yml` e um diretório chamado `conf` que contem um arquivo `my_proxy.conf`

**docker-compose.yml:** _Neste arquivo temos todas as configurações para subir o conteiner que será o proxy._
```
version: '3'

services:
  proxy-reverso:
    image: jwilder/nginx-proxy
    container_name: "proxy-reverso"
    restart: unless-stopped
    expose:
      - "80"
    network_mode: host
    volumes:
      - ./conf/my_proxy.conf:/etc/nginx/conf.d/my_proxy.conf:ro
      - /var/run/docker.sock:/tmp/docker.sock
#    command: [nginx-debug, '-g', 'daemon off;']

  whoami:
    image: jwilder/whoami
    environment:
      - VIRTUAL_HOST=whoami.local

```

**conf/my_proxy.conf:** _Este é o arquivo de configuração do proxy responsável pelo mapeamento de qual conteiner responderá por qual URL informada pelo cliente._
```
server {
        listen 80;
        server_name app1.acthosti.com.br;
        location / {
            proxy_pass http://127.0.0.1:8001;
            proxy_redirect off;
            proxy_bind 127.0.0.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Host $server_name;
            client_max_body_size 0;  
        }
}

server {
        listen 80;
        server_name dbapp1.acthosti.com.br;
        location / {
            proxy_pass http://127.0.0.1:8002;
            proxy_redirect off;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Host $server_name;
            client_max_body_size 0;
        }
}

server {
        listen 80;
        server_name app2.acthosti.com.br;
        location / {
            proxy_pass http://127.0.0.1:8003;
            proxy_redirect off;
            proxy_bind 127.0.0.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Host $server_name;
            client_max_body_size 0;
        }
}

server {
        listen 80;
        server_name dbapp2.acthosti.com.br;
        location / {
            proxy_pass http://127.0.0.1:8004;
            proxy_redirect off;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Host $server_name;
            client_max_body_size 0;
        }
}

```

**app1.acthosti.com.br:** A primeira aplicação responderá neste endereço. Aqui no meu exemplo este sera um wordpress.

**app2.acthosti.com.br:** A segunda aplicação responderá neste endereço. Aqui no meu exemplo este sera outro wordpress.

**dbapp1.acthosti.com.br:** Neste endereço estou configurando um PHPMyAdmin para acessar a base de dados primeira aplicação.

**dbapp2.acthosti.com.br:** Neste endereço estou configurando um PHPMyAdmin para acessar a base de dados da segunda aplicação.


### app1.acthosti.com.br
Neste diretório tudo que é necessário para configuração da primeira aplicação.

#### Entendendo a estrutura:
Neste diretório temos um arquivo `docker-compose.yml` e um diretório chamado `conf` que contem um arquivo `default.conf`

**docker-compose.yml:** _Neste arquivo temos todas as configurações para subir o conteiner da primeira aplicação. Aqui é apresentando todos os contêiners necessários a primeira aplicação, seus relacionamentos e mapeamento de volumes._
```
version: '3.3' 

services:

  db_internet_poupex:
    image: mariadb:latest
    container_name: "db_internet_poupex"
    volumes:
       - ./mariadb/conf.d:/etc/mysql/conf.d
       - ./mariadb/mysql.conf.d:/etc/mysql/mysql.conf.d
       - ./mariadb/data:/var/lib/mysql
       - ./mariadb/log:/var/log/
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: P@ssw0rd
      MYSQL_DATABASE: db_internet_poupex
      MYSQL_USER: db_internet_poupex
      MYSQL_PASSWORD: PS345px
 
  wordpress_internet_poupex:
    depends_on:
      - db_internet_poupex
    links:
      - db_internet_poupex
    image: wordpress:latest
    container_name: "wordpress_internet_poupex"
    restart: unless-stopped
    ports:
      - "8001:80"
    volumes:
      - ./wp-content/:/var/www/wordpress/
      - ./logs/:/var/log/
      - ./conf/etc/apache2/sites-enabled/:/etc/apache2/sites-enabled/:ro
    environment:
      WORDPRESS_DB_PREFIX: wp_poupex_
      WORDPRESS_DB_HOST: db_internet_poupex:3306
      WORDPRESS_DB_USER: db_internet_poupex
      WORDPRESS_DB_PASSWORD: PS345px
      WORDPRESS_DB_NAME: db_internet_poupex
      WORDPRESS_DB_TIMEOUT: 180

  phpmyadmin_internet_poupex:
    depends_on:
      - db_internet_poupex
    links:
      - db_internet_poupex:dbppx
    image: phpmyadmin/phpmyadmin:latest
    container_name: "phpmyadmin_internet_poupex"
    restart: unless-stopped
    ports:
      - "8002:80"
    environment:
      MYSQL_USERNAME: root
      MYSQL_ROOT_PASSWORD: P@ssw0rd

```


**conf/etc/apache2/sites-enabled/default.conf:** _Este é o arquivo de configuração do apache da primeira aplicação que determina a URL pela qual ele responderá, que neste caso é `app1.acthosti.com.br`._
```
<VirtualHost *:80>
        ServerName app1.acthosti.com.br

        #ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # LogLevel debug

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

```


### app2.acthosti.com.br
Neste diretório tudo que é necessário para configuração da primeira aplicação.

#### Entendendo a estrutura:
Neste diretório temos um arquivo `docker-compose.yml` e um diretório chamado `conf` que contem um arquivo `default.conf`

**docker-compose.yml:** _Neste arquivo temos todas as configurações para subir o conteiner da primeira aplicação. Aqui é apresentando todos os contêiners necessários a primeira aplicação, seus relacionamentos e mapeamento de volumes._
```
version: '3.3' 

services:

  db_internet_fhe:
    image: mariadb:latest
    container_name: "db_internet_fhe"
    volumes:
       - ./mariadb/conf.d:/etc/mysql/conf.d
       - ./mariadb/mysql.conf.d:/etc/mysql/mysql.conf.d
       - ./mariadb/data:/var/lib/mysql
       - ./mariadb/log:/var/log/
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: P@ssw0rd
      MYSQL_DATABASE: db_internet_fhe
      MYSQL_USER: db_internet_fhe
      MYSQL_PASSWORD: PS345fh
 
  wordpress_internet_fhe:
    depends_on:
      - db_internet_fhe
    links:
      - db_internet_fhe
    image: wordpress:latest
    container_name: "wordpress_internet_fhe"
    restart: unless-stopped
    ports:
      - "8003:80"
    volumes:
      - ./wp-content/:/var/www/wordpress/
      - ./logs/:/var/log/
      - ./conf/etc/apache2/sites-enabled/:/etc/apache2/sites-enabled/:ro
    environment:
      WORDPRESS_DB_PREFIX: wp_fhe_
      WORDPRESS_DB_HOST: db_internet_fhe:3306
      WORDPRESS_DB_USER: db_internet_fhe
      WORDPRESS_DB_PASSWORD: PS345fh
      WORDPRESS_DB_NAME: db_internet_fhe
      WORDPRESS_DB_TIMEOUT: 180

  phpmyadmin_internet_fhe:
    depends_on:
      - db_internet_fhe
    links:
      - db_internet_fhe:dbfhe
    image: phpmyadmin/phpmyadmin:latest
    container_name: "phpmyadmin_internet_fhe"
    restart: unless-stopped
    ports:
      - "8004:80"
    environment:
      MYSQL_USERNAME: root
      MYSQL_ROOT_PASSWORD: P@ssw0rd


```


**conf/etc/apache2/sites-enabled/default.conf:** _Este é o arquivo de configuração do apache da primeira aplicação que determina a URL pela qual ele responderá, que neste caso é `app2.acthosti.com.br`._
```
<VirtualHost *:80>
        ServerName app2.acthosti.com.br

        #ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # LogLevel debug

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

```

# Como utilizar esta solução?

## Exemplo
Para rodar este projeto, apenas siga os seguintes passos:


### 1)Entre no diretório 'proxy_reverso' e execute o comando `docker-compose up -d` para subir o cointeiner do NGINX

### 2)Entre no diretorio app1 e execute o comando `docker-compose up -d` para subir a primeira aplicação.

### 3)Entre no diretorio app2 e execute o comando `docker-compose up -d` para subir a segunda aplicação.

### Pronto! O serviço já deve estar sendo executado.


### Para parar a execução dos conteiners voce deve entrar em cada um dos diretórios que entrou para iniciar as execuções e digitar o comando `docker-compose stop`.
