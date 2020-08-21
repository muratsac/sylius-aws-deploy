<p align="center">
    <a href="https://sylius.com" target="_blank">
        <img height="75" src="https://demo.sylius.com/assets/shop/img/logo.png" />
    </a>
</p>
<p align="center">
    <a href="https://aws.amazon.com/tr/elasticbeanstalk/" target="_blank">
        <img height="75" src="https://avatars0.githubusercontent.com/u/2232217?s=200&v=4" />
    </a>
</p>

<h1 align="center">Sylius Deploy on AWS Elastic Beanstalk</h1>

### Platform
PHP 7.3 running on 64bit Amazon Linux 2

### Web Server
Nginx

## .ebextensions

### 1-packages.config

```yaml
packages:
  yum:
    gcc: []
    make: []
    htop: []
    php-mbstring: []
    php-gd: []
    php-pdo: []
    php-pear: []
    php-devel: []
    php-pecl-apcu: []

commands:
  01_install_node:
    command: |
      sudo curl --silent --location https://rpm.nodesource.com/setup_13.x | sudo bash -
      sudo yum -y install nodejs

  02_install_yarn:
    command: |
      sudo wget https://dl.yarnpkg.com/rpm/yarn.repo -O /etc/yum.repos.d/yarn.repo
      sudo yum -y install yarn
```

### 2-composer.config

```yaml
commands:
  01_update_composer:
    command: export COMPOSER_HOME=/root && /usr/bin/composer.phar self-update

option_settings:
  - namespace: aws:elasticbeanstalk:application:environment
    option_name: COMPOSER_HOME
    value: /root
  - namespace: aws:elasticbeanstalk:container:php:phpini
    option_name: document_root
    value: /public
  - namespace: aws:elasticbeanstalk:container:php:phpini
    option_name: composer_options
    value: --optimize-autoloader --ignore-platform-reqs --no-ansi --no-interaction --no-progress --no-suggest
```

### 3-app.config

```yaml
container_commands:
  1_writable_directories:
    command: sudo chmod -R 777 /var/app/staging/var && chmod -R 777 /var/app/staging/public
  2_dugun_db_migration:
    command: cd /var/app/staging && php bin/console doctrine:migrations:migrate --no-interaction
  3_doctrine_cache_clear:
    command: cd /var/app/staging &&  php bin/console doctrine:cache:clear-metadata && sudo php bin/console doctrine:cache:clear-query
  4_install_assets:
    command: cd /var/app/staging &&  php bin/console sylius:install:assets --no-interaction
  5_nginx_reload:
     command: sudo systemctl restart nginx
  6_yarn_dependency:
     command: cd /var/app/staging && yarn install
  7_yarn_frontend:
     command: cd /var/app/staging && yarn build
  8_cache_warmup:
    command: HTTPDUSER=$(ps axo user,comm | grep -E '[a]pache|[h]ttpd|[_]www|[w]ww-data|[n]ginx|webapp' | grep -v root | head -1 | cut -d\  -f1) && sudo setfacl -dR -m u:"$HTTPDUSER":rwX -m u:$(whoami):rwX var && sudo setfacl -R -m u:"$HTTPDUSER":rwX -m u:$(whoami):rwX var
```

## .platform

### Override default php.conf (nginx/conf.d/elasticbeanstalk/php.conf)

```yaml
root /var/app/current/public;

index index.php;

location / {
    # try to serve file directly, fallback to index.php
    try_files $uri /index.php$is_args$args;
}

location ~ \.(php|phar)(/.*)?$ {
    fastcgi_split_path_info ^(.+\.(?:php|phar))(/.*)$;

    fastcgi_intercept_errors on;
    fastcgi_index  index.php;

    fastcgi_param  QUERY_STRING       $query_string;
    fastcgi_param  REQUEST_METHOD     $request_method;
    fastcgi_param  CONTENT_TYPE       $content_type;
    fastcgi_param  CONTENT_LENGTH     $content_length;

    fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
    fastcgi_param  REQUEST_URI        $request_uri;
    fastcgi_param  DOCUMENT_URI       $document_uri;
    fastcgi_param  DOCUMENT_ROOT      $document_root;
    fastcgi_param  SERVER_PROTOCOL    $server_protocol;
    fastcgi_param  REQUEST_SCHEME     $scheme;
    fastcgi_param  HTTPS              $https if_not_empty;

    fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
    fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;

    fastcgi_param  REMOTE_ADDR        $remote_addr;
    fastcgi_param  REMOTE_PORT        $remote_port;
    fastcgi_param  SERVER_ADDR        $server_addr;
    fastcgi_param  SERVER_PORT        $server_port;
    fastcgi_param  SERVER_NAME        $server_name;

    # PHP only, required if PHP was built with --enable-force-cgi-redirect
    fastcgi_param  REDIRECT_STATUS    200;

    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    fastcgi_param  PATH_INFO $fastcgi_path_info;
    fastcgi_pass   php-fpm;
}
```