# add: Setting
```bash:bash
$ touch {Dockerfile, docker-compose.yml, Gemfile, Gemfile.lock, entrypoint.sh}
```
#### Dockerfile
```dockerfile:dockerfile
FROM --platform=amd64 ruby:3.3.5

RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
  && echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
  && apt-get update \
  && curl -fsSL https://deb.nodesource.com/setup_14.x | bash \
  && apt-get install -y nodejs npm cron \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* \
  && mkdir /sample_app

RUN npm install --global yarn

WORKDIR /sample_app
COPY Gemfile /sample_app/Gemfile

COPY Gemfile.lock /sample_app/Gemfile.lock

RUN bundle install

COPY . /sample_app

COPY entrypoint.sh /usr/bin/

RUN chmod +x /usr/bin/entrypoint.sh

ENTRYPOINT ["entrypoint.sh"]

EXPOSE 3000

CMD ["rails", "server", "-b", "0.0.0.0"]
```



#### docker-compose.yml
```
services:
  mysql:
    platform: linux/x86_64
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
    ports:
      - "3310:3306"
    command: --default-authentication-plugin=mysql_native_password
    volumes:
      - mysql_data3:/var/lib/mysql
  sample_app:
    build: .
    command: /bin/sh -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    environment:
      MYSQL_DATABASE: sample_app_db
      MYSQL_HOST: mysql
      MYSQL_USERNAME: root
      MYSQL_ROOT_PASSWORD: password
      MYSQL_PORT: 3306
    volumes:
      - .:/sample_app
      - rails_gem_data:/usr/local/bundle
    ports:
      - "3003:3000"
    depends_on:
      - mysql
    stdin_open: true
    tty: true

volumes:
  mysql_data3:
  rails_gem_data:
    driver: local
```



#### Gemfile
```
source "https://rubygems.org"

ruby "3.3.5"

gem "rails", "~> 7.2.1"
```



#### Gemfile.lock
```
こちらは、準備だけしてからで。
```



#### entrypoint.sh
```
#!/bin/bash
set -e

# Remove a potentially pre-existing server.pid for Rails.
rm -f /myapp/tmp/pids/server.pid

# Then exec the container's main process (what's set as CMD in the Dockerfile).
exec "$@"
```



##### Railsアプリ作成
```
$ docker compose build
$ docker compose run --rm sample_app rails new . --force --database=mysql -j esbuild
```
ここで ***database.yml*** を編集する
```
default: &default
  adapter: mysql2
  encoding: utf8mb4
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: root
  password: password　<--記載されてなかったのでpasswordを追加
  host: mysql <--compose.ymlと同じmysqlに変更する
  port: 3306  <--デフォルトポートを追加する
```
```
$ docker compose run --rm sample_app bundle exec rails db:create
```
***Procfile.dev*** の ***RUBY_DEBUG_OPEN=true bin/rails server*** を削除する。
###### 理由
> bin/dev コマンドは foreman と呼ばれるプロセス管理ツールを使っています。これで Rails サーバーを動かすと binding.pry などで止めたときに標準入力ができず、デバッグすることができません。
なので、Rails サーバーは直接動かしつつ、CSS と JavaScript は foreman に動かしてもらうようにします。



####ターミナルを立ち上げる
cssとjs用のbin/devとrailsを立ち上げる
***localhost:3003***
```
docker compose run --rm sample_app bin/dev
docker compose run --rm --service-ports sample_app
```


#### TailWindCSSをinstall
```
docker compose --rm sample_app bundle add tailwindcss-rails
docker compose --rm sample_app bundle exec rails tailwindcss:install
```
