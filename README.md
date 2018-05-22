# Travis-Heroku-Docker

Check related app repo [here](https://github.com/blarvin/TEAM-MALN-ACEBOOK)

This a tutorial that covers the set-up environment involving Travis CI - Heroku - Docker for a Ruby on Rails project.
Also, this tutorial will show how to overcome common difficulties that you can be faced with during set-up.

## Part I. Set up.

### Travis CI

* Go to [Travis](https://travis-ci.org/) and log in with your GitHub credentials.
* Follow the instructions on Travis about how to set-up a new repository.
* Make sure that in the root of your repository you have ```.travis.yml``` file.

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

* Go to [Heroku](https://dashboard.heroku.com) and make a free account
* We assume that you have Ruby installed

In your terminal (you should be in your repo):

* ```$ heroku login``` 
* ```$ heroku create```
* ```$ git push heroku master```
* ```$ heroku open```

At this stage, you should be able to see your app opened with Heroku. If you are facing any difficulties, you can see your logs and follow a debugging process: ```$ heroku logs --tail```

### Docker

* Go to [Docker](https://www.docker.com/) and make an account.
* Make sure that you have the following files inside your repo:

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

#### Dockerfile

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
  localhost: db    #You need this line to build app with docker, but this line should be deleted before you push to Github, otherwise, Travis would fail.
test:
<<: *default
database: pgapp_test
localhost: db     #You need this line to build app with docker, but this line should be deleted before you push to Github, otherwise, Travis would fail.

production:
  <<: *default
  host: <%= ENV['DATABASE_HOST'] %>
  database: <%= ENV['DATABASE_NAME'] %>
  username: <%= ENV['DATABASE_USER'] %>
  password: <%= ENV['DATABASE_PASSWORD'] %>
```

### Deploy via Heroku using Docker

In your terminal:

```
$ docker login
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

#### CMD bundle exec rackup --port=$PORT

Basically, our first mistake was that we did not know that we should specify which port should be used in our Dockerfile. At first, we were guessing and tried to change random lines of code in config to make it work. We quickly realized that this approach would not work, we forced ourselves to go through a proper debugging process:
* raise visibility --> using logs --> google errors --> discuss with pair partner if it makes sense to change this particular part of the code.
It helped us to go through process quickly and step by step we reached a point where we can run our app locally with docker, but it was broken on Heroku. We could not figure out what went wrong and followed our agreement to not be stuck for too long, so we asked a coach for some assistance.
Together we found out how to modify our config file and added the following line ```CMD bundle exec rackup --port=$PORT```.

#### Removing pid files

If you have a mistake like the following: ``` A server is already running. Check /maln-acebook/tmp/pids/server.pid.```. You should run following comands in your terminal:
* ```$ sudo rm tmp/pids/server.pid```
* ```$ docker-compose run web rake db:reset```
* ```$ docker-compose up --build```

#### Database issues

The relationship between Docker and Heroku require you to do the following:
* Every time when you are introducing a new ```gem``` to your app you have to run ```$ docker-compose run web bundle install```
* Every time you change the logic in the database you need to drop heroku database:
1. ```$ heroku pg:reset DATABASE_URL -a maln-acebook``` and ```--confirm``` this command
2. Remove pids
3. ```$ heroku container:login```
4. ```$ heroku container:push web -a maln-acebook```
