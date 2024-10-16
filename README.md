# Tutorial de deploy do Django

Curso Django Pro: Do zero ao Deploy -> [Acesse o curso Django Pro](https://jeffelobato.com/curso-django-pro)


- Criar um diretório myblog para aplicação dentro de /var/www/
- Dar um git init, criar repositório remoto e fazer um pull da aplicação dentro de myblog
- Dar permissões www-data ao diretório myblog

## Como instalar a aplicação?

- Instalar o MySql-Server e configurar um usuário e senha

```Shell
sudo apt-get update

sudo apt-get install mysql-server

sudo mysql -u root

ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'YOUR_PASSWORD';

exit

sudo mysql_secure_installation

# Sobre as configurações do mysql_secure_installation
"""
1. Validar a configuração do plugin de senha?  
    Recomendação: Sim (Y).  
    Este plugin ajuda a definir a política de complexidade das senhas, garantindo que as senhas usadas sejam fortes.
    Escolha ao menos a opção de nível médio para novas senhas.
    
2. Alterar a senha do usuário root?  
    Recomendação: Sim (Y).  
    Sempre é recomendável definir uma senha forte para o usuário `root`, caso ainda não tenha sido definida. Utilize uma combinação de letras maiúsculas, minúsculas, números e caracteres especiais.
    
3. Remover usuários anônimos?  
    Recomendação: Sim (Y).  
    Usuários anônimos podem ser uma vulnerabilidade de segurança. É melhor removê-los para restringir o acesso ao banco de dados.
    
4. Desabilitar o login remoto do root?  
    Recomendação: Sim (Y).  
    Desabilitar o acesso remoto do `root` é uma medida de segurança importante, pois evita que um atacante tente acessar o banco de dados a partir de outra máquina. Se precisar de acesso remoto, use um usuário menos privilegiado.
    
5. Remover o banco de dados de teste e acesso a ele?  
    Recomendação: Sim (Y).  
    O banco de dados de teste é criado por padrão e pode ser acessado por qualquer usuário. É melhor removê-lo se não for necessário.
    
6. Recarregar as tabelas de privilégios agora?  
    Recomendação: Sim (Y).  
    Isso aplica imediatamente todas as mudanças feitas, garantindo que as novas configurações de segurança entrem em vigor.
"""

sudo mysql -u root -p

create user 'nomeDoUsuario'@'localhost' identified by 'SENHA';
create database nomeDoBancoDeDados;

grant all privileges on nomeDoBancoDeDados.* to 'nomeDoUsuario'@'localhost';
```
Obs.: Ao criar o usuário, colocar '%' depois do @ se o acesso for por outro host, localhost apenas se for local.

- Instalar mysqlclient alguns pocotes importantes do python
```Shell
sudo apt install libmysqlclient-dev
sudo apt install pkg-config
sudo apt install build-essential
sudo apt install python3-dev
```
 
- Instar o venv do python
```bash
apt-get install python3-venv
```

- Dentro do diretório do projeto onde se encontra o arquivo manage.py, criar um ambiente virtual python (talvez seja necessário instalar o venv do python)
	```shell
python3 -m venv env

	```
	
- Entrar no ambiente virtual
	```shell
source env/bin/activate
	```
	
- Instalar os pacotes do arquivo requirements.txt
	```shell
pip install -r requirements.txt
# talvez precise antes de tudo instalar o pip com sudo apt-get install pip
	```
	
- Instalar o mysqlclient com o pip dentro do ambiente virtual caso não esteja no requirements.txt
```Shell
pip install mysqlclient
```

- Fazer o makemigrations e migrate
	```python
python manage.py makemigrations
python manage.py migrate
	```

- Crie um arquivo .env com todas as variáves de ambiente que serão utilizadas na aplicação

- Criar uma chave aleatória para colocar no .env do projeto DJango
	```Shell
python manage.py shell
>>> from django.core.management.utils import get_random_secret_key
>>> get_random_secret_key()
#Copiar a chave e colocar no arquivo .env na variável DJANGO_KEY

>>> exit()
	```
	Obs.: Caso não esteja utilizando um .env substitua a secret key do django direto no arquivo de configuração.

- Crie um novo superuser, se sua aplicação precisar:
```shell
 python manage.py createsuperuser
```

- Instalar o uWSGI e outros pacotes:

```Shell
sudo apt install libssl-dev libffi-dev python-dev-is-python3

# Caso não esteja no requirementos.txt
# Lembre de estar na venv para instalar esses pacotes
pip install wheel
pip install uwsgi
```

- Instalar Nginx

```Shell
sudo apt install nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

- Criar um arquivo uwsgi_params dentro do diretório do projeto, mesmo diretório que tem o arquivo manage.py

```Shell
nano uwsgi_params

uwsgi_param  QUERY_STRING       $query_string;
uwsgi_param  REQUEST_METHOD     $request_method;
uwsgi_param  CONTENT_TYPE       $content_type;
uwsgi_param  CONTENT_LENGTH     $content_length;

uwsgi_param  REQUEST_URI        $request_uri;
uwsgi_param  PATH_INFO          $document_uri;
uwsgi_param  DOCUMENT_ROOT      $document_root;
uwsgi_param  SERVER_PROTOCOL    $server_protocol;
uwsgi_param  REQUEST_SCHEME     $scheme;
uwsgi_param  HTTPS              $https if_not_empty;

uwsgi_param  REMOTE_ADDR        $remote_addr;
uwsgi_param  REMOTE_PORT        $remote_port;
uwsgi_param  SERVER_PORT        $server_port;
uwsgi_param  SERVER_NAME        $server_name;
```

- Arquivo de configuração do NGINX (Substitua myblog pelo nome do diretório da sua aplicação, se for o caso)

```Shell
nano /etc/nginx/sites-available/myblog_nginx.conf

upstream django {
    server unix:///var/www/myblog/mysite.sock; 
}

server {
    listen      443;
    server_name DOMÍNIO ou IP;
    charset     utf-8;

    client_max_body_size 75M; #Coloque o tamanho máximo em MB que você considera que a requisição da sua aplicação pode ter 

    location /media  {
        alias /var/www/myblog/media; 
    }

    location /static {
        alias /var/www/myblog/staticfiles;
    }

    location / {
        uwsgi_pass  django;
        include     /var/www/myblog/uwsgi_params; 
    }
}

server {
    listen      80;
    server_name DOMÍNIO ou IP;
    charset     utf-8;

    client_max_body_size 75M; #Coloque o tamanho máximo em MB que você considera que a requisição da sua aplicação pode ter 

    location /media  {
        alias /var/www/myblog/media; 
    }

    location /static {
        alias /var/www/myblog/staticfiles;
    }

    location / {
        uwsgi_pass  django;
        include     /var/www/myblog/uwsgi_params; 
    }
}

```
Obs.: Aqui estamos levando em consideração que o certificado SSL será o do CloudFlare, caso utilize outro tipo de certificado, é provável que o tenho que configurar aqui nesse arquivo, outro detalhe é que é interessante que seja feito um redirecionamento automático da porta 80 para o https na porta 443, mas neste caso o redirecionamento também será feito pelo CloudFlare.

Outras configurações importantes como cabeçalho HSTS também são importantes, é possível fazer tanto nesse arquivo, quanto direto no settings do Django, mas também estamos considerando que isso será feito pelo CloudFlare, logo, então, não é necessário configurar.

- Criar um symlink em sites-enabled
```Shell

sudo rm -rf /etc/nginx/sites-enabled/default

ln -s /etc/nginx/sites-available/myblog_nginx.conf /etc/nginx/sites-enabled/
# substitua o nome do arquivo de configuração pelo nome correto, se for o caso.
```

- Resetar o Nginx
``` Shell

sudo /etc/init.d/nginx restart

```

- Se quiser testar o uWSGI (substitua my_blog pelo nome do diretório do seu projeto que possui o arquivo wsgi se for o caso)
```Shell
# Rodar dentro da venv, comando dentro da pasta myblog

uwsgi --socket mysite.sock --module my_blog.wsgi --chmod-socket=666

```

- Criar um arquivo ini dentro do diretório do projeto, onde tem o arquivo manage.py (substitua my_blog pelo nome do diretório do seu projeto que possui o arquivo wsgi e modificar o nome env para o nome da sua env, se for o caso)

```Shell

nano /var/www/myblog/myblog_uwsgi.ini

[uwsgi]
chdir           = /var/www/myblog
module          = my_blog.wsgi
home            = /var/www/myblog/env
master          = true
processes       = 10
socket          = /var/www/myblog/mysite.sock
vacuum          = true
chmod-socket    = 666

```

- Se quiser testar o arquivo ini
```Shell

uwsgi --ini myblog_uwsgi.ini

```

 - Configurando o uWSGI em modo Emperor (Gerenciamento de instâncias)

```Shell

sudo mkdir /etc/uwsgi
sudo mkdir /etc/uwsgi/vassals
sudo ln -s /var/www/myblog/myblog_uwsgi.ini /etc/uwsgi/vassals/
uwsgi --emperor /etc/uwsgi/vassals --uid www-data --gid www-data

```

 - Criar o arquivo .service para automatizar o início e restart do serviço uwsgi (substitua my_blog pelo nome do diretório do seu projeto que possui o arquivo wsgi e modificar o nome env para o nome da sua env, se for o caso)
```Shell

nano /etc/systemd/system/myblog_uwsgi.service


[Unit]
Description=Django VPS uWSGI Emperor
After=syslog.target

[Service]
ExecStart=/var/www/myblog/env/bin/uwsgi --emperor /etc/uwsgi/vassals --uid www-data --gid www-data
RuntimeDirectory=uwsgi
Restart=always
KillSignal=SIGQUIT
Type=notify
StandardError=syslog
NotifyAccess=all
User=www-data

[Install]
WantedBy=multi-user.target

```

- Dar permissões e ativar o serviço
```Shell

sudo chmod 664 /etc/systemd/system/myblog_uwsgi.service

sudo systemctl daemon-reload

sudo systemctl enable myblog_uwsgi.service

sudo systemctl start myblog_uwsgi.service

# Se quiser ver o status do serviço
sudo systemctl status myblog_uwsgi.service

# Se quiser ver os logs do serviço
journalctl -u djangovps.service

```


# Atualização da aplicação

## Atualização

- Caso tenha alguma atualização no repositório, para atualizar no servidor, basta ir até a pasta da aplicação Prontuario pelo terminal e dar um git pull
```Shell
cd /var/www/myblog/

git pull origin main

source ./env/bin/activate

python manage.py makemigrations

python manage.py migrate

python manage.py collectstatic
```

- Reinicie o uWSGI e o Nginx sempre que fizer uma atualização na sua aplicação

```Shell
sudo systemctl restart myblog_uwsgi.service
sudo systemctl restart nginx
```


