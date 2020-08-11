# Sylius deploy on AWS Elastic Beanstalk

### Platform
PHP 7.3 running on 64bit Amazon Linux 2

### Web Server
Nginx

## .ebextensions

### 1-yarn.config

```yaml
commands:
  01_install_node:
    command: |
      sudo curl --silent --location https://rpm.nodesource.com/setup_8.x | sudo bash -
      sudo yum -y install nodejs

  02_install_yarn:
    # don't run the command if yarn is already installed (file /usr/bin/yarn exists)
    test: '[ ! -f /usr/bin/yarn ] && echo "Yarn not found, installing..."'
    command: |
      sudo wget https://dl.yarnpkg.com/rpm/yarn.repo -O /etc/yum.repos.d/yarn.repo
      sudo yum -y install yarn
```

### 2-composer.config

```yaml
option_settings:
  aws:elasticbeanstalk:container:php:phpini:
    document_root: /public
```

### 3-app.config

```yaml
container_commands:
  1_writable_directories:
    command: sudo chmod -R 777 /var/app/current/var && chmod -R 777 /var/app/current/public
  2_dugun_db_migration:
    command: php bin/console doctrine:migrations:migrate --no-interaction
  3_doctrine_cache_clear:
    command: php bin/console doctrine:cache:clear-metadata && sudo php bin/console doctrine:cache:clear-query
  4_nginx_reload:
     command: sudo systemctl restart nginx
  5_yarn_install:
     command: yarn install
  6_yarn_build:
     command: yarn build
  7_cache_warmup:
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