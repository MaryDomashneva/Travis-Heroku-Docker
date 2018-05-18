# Environment-set-up-Travis-Heroku_Docker

This a tutorial that covered set-up environment Travis CI - Heroku - Doker for Ruby on rails project.
Also, this tutorial will show how to overcome common difficulties that you can face during set-up.

## Part I. Set up.

### Travis CI

* Go to https://travis-ci.org/ and log in with your GitHub.
* Follow the instructions on Travis about how to set-up a new repository.
* Make sure, that in the root of your repository you have ```.travis.yml``` file.

#### .travis.yml

```
language: ruby
rvm:
- 2.5.0
script:
- bundle exec rspec spec
services:
- postgresql
before_install:
- gem install bundler ### this line is important to have for further operations with Heroku and docker
before_script:
- bin/rails db:create
- bin/rails db:migrate
deploy:
  provider: Heroku
  api_key:
    secure: ### HERE SHOULD BE YOUR KEYS (YOU CAN GET THEM BY TYPING IN TERMINAL $ setup heroku --force)
  app: maln-acebook
  on:
    repo: blarvin/TEAM-MALN-ACEBOOK ### YOUR REPO IS HERE
```
### Heroku

* Go to https://dashboard.heroku.com and make a free account
* We assume that you have Ruby installed

In your terminal (you should be in your repo):

* ```$ heroku login``` 
* ```$ heroku create```
* ```$ git push heroku master```
* ```$ heroku open```

On this stage, you should be able to see your app opened with Heroku. If you facing any difficulties you can every time see your logs and following debugging process: ```$ heroku logs --tail```

### Docker

* Go to https://www.docker.com/ and make an account.
* Make sure that you have inside you repo following files:

#### docker-compose.yml

```
version: '3'
services:
  db:
    image: postgres
    volumes:
      - ./tmp/db:/var/lib/postgresql/data
  web:
    build: .
    command: bundle exec rails s -p 3000 -b '0.0.0.0'
    volumes:
      - .:/maln-acebook
    ports:
      - "3000:3000"
    depends_on:
      - db
```

#### Dokerfile

```
FROM ruby:2.5.0
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev nodejs
RUN mkdir /maln-acebook
WORKDIR /maln-acebook
COPY Gemfile /maln-acebook/Gemfile
COPY Gemfile.lock /maln-acebook/Gemfile.lock
RUN bundle install
COPY . /maln-acebook

CMD bundle exec rackup --port=$PORT  #This is very important line. Do not forget to include it, otherwise, you docker would not find localhost
```
In configs:

#### database.yml

```
default: &default
  adapter: postgresql
  encoding: unicode
  username: postgres
  password:
  pool: 5

development:
  <<: *default
  database: pgapp_development
  localhost: db    #You need this line to build app with docker, but this lines should be deleted before you push to Github, otherwise, Travis would fail.
test:
<<: *default
database: pgapp_test
localhost: db     #You need this line to build app with docker, but this lines should be deleted before you push to Github, otherwise, Travis would fail.

production:
  <<: *default
  host: <%= ENV['DATABASE_HOST'] %>
  database: <%= ENV['DATABASE_NAME'] %>
  username: <%= ENV['DATABASE_USER'] %>
  password: <%= ENV['DATABASE_PASSWORD'] %>
```

### Deploy Heroku using Docker

In your terminal:

```
$ docker-compose run web bundle install

$ docker-compose run web rake db:create # create databases
$ docker-compose run web rake db:migrate # create tables
$ docker-compose run web rake db:seed # insert seed records

$ docker-compose up --build # go to localhost:3000/api/users in your browser or `curl localhost:3000/api/users`

$ docker-compose ls # to show current containers
$ docker-compose down # to stop the container
```

```
$ heroku container:login
$ heroku container:push web -a maln-acebook
```

## Part II. Difficulties.
