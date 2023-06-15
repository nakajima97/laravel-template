# 概要
Laravelで開発を行う際のテンプレートリポジトリ

# git clone後に実行するコマンド
```
docker run --rm \
    -u "$(id -u):$(id -g)" \
    -v $(pwd):/var/www/html \
    -w /var/www/html \
    laravelsail/php81-composer:latest \
    composer install --ignore-platform-reqs
```

# 主要パッケージ
- Laravel sail
- Larastan

# Larastanの実行
'./vendor/bin/phpstan analyse'

# PHP CS Fixerの実行
'tools/php-cs-fixer/vendor/bin/php-cs-fixer fix app'
