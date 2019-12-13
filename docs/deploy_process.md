# Autodeploy proccess

In this section you'll learn how to make autodeploy *__project__* with *__Gitlab CI/CD__*.

*__How it works?__*

After pushing in branch, your code start deploying on server automaticaly and you don’t have to make movements. All will be automaticaly.

It will be work with help of `gitlab pipelines`.

Pipelines are the top-level component of continuous integration, delivery, and deployment.

Pipelines comprise:

1. Jobs that define what to run. For example, code compilation or test runs.
2. Stages that define when and how to run. For example, that tests run only after code compilation.

The action chain looks like this:

1. You push new changes (*__then gitlab pipelines do everything automaticaly__*).
2. Code build with docker.
3. Connect to the server with SSH.
4. Deploy on the server.


First things first, you need your project with **docker** on **gitlab**. How to *__setup docker__* check another documentation.

Then you need to create file `.gitlab-ci.yml` in root folder of project or create on start page of your gitlab repository *__button setup CI/CD__*.

We'll make `DIND` (Docker in docker).

At start Gitlab CI has one docker image (_in example docker:19:03.1_).
And in this image we will push our images.

Here’s a simple `.gitlab-ci.yml` file showing how Docker caching can be utilized:

``` diff
image: docker:19.03.1

services:
  - docker:19.03.1-dind

variables:
  # Use TLS https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#tls-enabled
  DOCKER_HOST: tcp://docker:2376
  DOCKER_TLS_CERTDIR: "/certs"

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:
  stage: build
  script:
    - docker pull $CI_REGISTRY_IMAGE:latest || true
    - docker build --cache-from $CI_REGISTRY_IMAGE:latest --tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA --tag $CI_REGISTRY_IMAGE:latest .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
```

The steps in the `script` section for the `build` stage can be summed up to:

1. The first command tries to pull the image from the registry so that it can be used as a cache for the `docker build` command.
2. The second command builds a Docker image using the pulled image as a cache (notice the `--cache-from $CI_REGISTRY_IMAGE:latest` argument) if available, and tags it.
3. The last two commands push the tagged Docker images to the container registry so that they may also be used as cache for subsequent builds.

After adding file `.gitlab-ci.yml` and after every push on git, our pipeline will start.

The result: ![gitlab-ci.png](attachments/gitlab-ci.png)

Pipeline has one from 4 states:

``` diff
! 1. Pending
+ 2. Passed
# 3. Canceled
- 4. Failed
```

Before we'll start our pipelines, we need to set `VARIABLES`

![variables.png](attachments/variables.png)

## Insert values by example

``` diff
# All this variables should be set in GitLab CI/CD settings for deployment,
# APP_HOST should be your public DNS
# Remove all comments on the end of the lines to run with docker

# environment
APP_HOST= your_server_example.com # without http:// or localhost for docker
RAILS_ENV=production
RACK_ENV=production
CI_DEPLOY_USER=gitlab+deploy-token-111111 # check  under this example how to create
CI_DEPLOY_PASSWORD=password
DB_HOST=localhost # db to run in docker
DB_PASSWORD=''
DB_USERNAME=postgres
REDIS_URL=redis://redis:6379/0
RAILS_SERVE_STATIC_FILES=enabled
RAILS_LOG_TO_STDOUT=enabled
SECRET_KEY_BASE=400c6ec2a685f346f0718301a4d8eeec0080c15278c984f8d63849bbfba3dfd30e140342cc3406eb79581acecf635b92185a4d8ba68ca533aacbe1fc5cde775d
JWT_KEY=secret
SSH_PRIVATE_KEY=-----BEGIN RSA PRIVATE KEY--------- # our private key from server
# smtp settings
SMTP_ADDRESS=smtp.gmail.com
SMTP_DOMAIN=gmail.com
SMTP_PORT=587
GMAIL_OWNER=your_email@gmail.com
GMAIL_PASSWORD=your_password

# frontend
VUE_APP_BASE_HOST=/ # relative path for API requests from frontend
```

Take your .ENV file and set **Variables** one by one:

We must set SSH_PRIVATE_KEY as variable from key-file ssh (`.pem file or rsa-file`)

![variables-all.png](attachments/variables_all.png)

## Deploy tokens

Also you should add deploy token to GitLab variables

Deploy tokens allow to download `(through git clone)`, or read the container registry images of a project without the need of having a user and a password.

Please note, that the expiration of deploy tokens happens on the date you define, at midnight UTC and that they can be only managed by `maintainers` (check your role on a project).

## Creating a Deploy Token

You can create as many deploy tokens as you like from the settings of your project:

1. Log in to your GitLab account.
2. Go to the project you want to create Deploy Tokens for.
3. Go to *__Settings > Repository__*.
4. Click on “Expand” on *__Deploy Tokens__* section.
5. Choose a name, expiry date (optional), and username (optional) for the token.
6. Choose the desired scopes.
7. Click on *__Create deploy token__*.
8. Save the deploy token somewhere safe. Once you leave or refresh the page, *__you won’t be able to access it again__*.

![deploy_tokens.png](attachments/deploy_tokens.png)

### Deploy token custom username

The default username format is `gitlab+deploy-token-#{n}`. Some tools or platforms may not support this format, in such case you can specify custom username to be used when creating the deploy token.

There’s a special case when it comes to Deploy Tokens. If a user creates one named `gitlab-deploy-token`, the username and token of the Deploy Token will be automatically exposed to the CI/CD jobs as environment variables: `CI_DEPLOY_USER` and `CI_DEPLOY_PASSWORD`, respectively. With the GitLab Deploy Token, the `read_registry` scope is implied.

After you create the token, you can login to the Container Registry using those variables:

``` diff
docker login -u $CI_DEPLOY_USER -p $CI_DEPLOY_PASSWORD $CI_REGISTRY
```

`ubuntu` - is it user on your server.

The latest file `.gitlab-ci.yml`:

``` diff
stages:
  - build
  - deploy

build-master:
  stage: build
  only:
    - master
  image:
    name: docker/compose:1.21.2
    entrypoint: ["/bin/sh", "-c"]
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
    NGINX_PORT: "80:80"
    DATABASE_PORT: "5433:5432"
    APP_HOST_PORT: "${APP_HOST}:80"
  services:
    - docker:18.09-dind
  script:
    - docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_REGISTRY
    - env | grep -E 'DB_|REDIS_URL|RACK_ENV|RAILS_|SECRET_KEY_BASE|SMTP_|GMAIL_|VUE_APP_BASE_HOST|NGINX_PORT|APP_HOST|DATABASE_PORT|APP_HOST_PORT' > .env
    - docker-compose build
    - docker-compose push

build-dev:
  stage: build
  only:
    - dev
  image:
    name: docker/compose:1.21.2
    entrypoint: ["/bin/sh", "-c"]
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
    NGINX_PORT: "81:80"
    DATABASE_PORT: "5434:5432"
    APP_HOST_PORT: "${APP_HOST}:81"
  services:
    - docker:18.09-dind
  script:
    - docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_REGISTRY
    - env | grep -E 'DB_|REDIS_URL|RACK_ENV|RAILS_|SECRET_KEY_BASE|SMTP_|GMAIL_|VUE_APP_BASE_HOST|NGINX_PORT|APP_HOST|DATABASE_PORT|APP_HOST_PORT' > .env
    - docker-compose build
    - docker-compose push

build-branches:
  stage: build
  except:
    - dev
    - master
  image:
    name: docker/compose:1.21.2
    entrypoint: ["/bin/sh", "-c"]
  before_script:
    - export RANDOM_PORT=$(shuf -i 82-89 -n 1)
    - export RANDOM_PORT_DB=$(shuf -i 5434-5439 -n 1)
    - export NGINX_PORT="${RANDOM_PORT}:80"
    - export DATABASE_PORT="${RANDOM_PORT_DB}:5432"
    - export APP_HOST_PORT="${APP_HOST}:${RANDOM_PORT}"
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
  services:
    - docker:18.09-dind
  script:
    - docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_REGISTRY
    - env | grep -E 'DB_|REDIS_URL|RACK_ENV|RAILS_|SECRET_KEY_BASE|SMTP_|GMAIL_|VUE_APP_BASE_HOST|NGINX_PORT|APP_HOST|DATABASE_PORT|APP_HOST_PORT|RANDOM_PORT|RANDOM_PORT_DB' > .env
    - docker-compose build
    - docker-compose push
deploy-production:
  stage: deploy
  only:
    - master
  image: alpine:latest
  before_script:
    - hostname
    - pwd
    - whoami
    - apk update && apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
  variables:
    NGINX_PORT: "80:80"
    DATABASE_PORT: "5433:5432"
    APP_HOST_PORT: "${APP_HOST}:80"
  script:
    - env | grep -E 'CI_REGISTRY_IMAGE|CI_COMMIT_SHORT_SHA|DB_|REDIS_URL|RACK_ENV|RAILS_|SECRET_KEY_BASE|SMTP_|GMAIL_|VUE_APP_BASE_HOST|APP_HOST|NGINX_PORT|DATABASE_PORT|APP_HOST_PORT' > .env
    - chmod 744 docker-compose.yml
    - chmod 744 .env
    - ssh -T ubuntu@$APP_HOST "mkdir -p ~/apps/production"
    - ssh -T ubuntu@$APP_HOST "chmod 777 ~/apps/production"
    - scp docker-compose.yml ubuntu@$APP_HOST:~/apps/production
    - scp .env ubuntu@$APP_HOST:~/apps/production
    - ssh -T ubuntu@$APP_HOST "docker login -u $CI_DEPLOY_USER -p $CI_DEPLOY_PASSWORD $CI_REGISTRY && cd apps/production && docker-compose pull && docker-compose up -d"
    - ssh -T ubuntu@$APP_HOST "cd apps/production && docker-compose run --rm web rake db:create db:migrate db:seed && docker-compose up -d"


deploy-development:
  stage: deploy
  only:
    - dev
  image: alpine:latest
  before_script:
    - hostname
    - pwd
    - whoami
    - apk update && apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
  variables:
    NGINX_PORT: "81:80"
    DATABASE_PORT: "5434:5432"
    APP_HOST_PORT: "${APP_HOST}:81"
  script:
    - env | grep -E 'CI_REGISTRY_IMAGE|CI_COMMIT_SHORT_SHA|DB_|REDIS_URL|RACK_ENV|RAILS_|SECRET_KEY_BASE|SMTP_|GMAIL_|VUE_APP_BASE_HOST|APP_HOST|NGINX_PORT|DATABASE_PORT|APP_HOST_PORT' > .env
    - chmod 744 docker-compose.yml
    - chmod 744 .env
    - ssh -T ubuntu@$APP_HOST "mkdir -p ~/apps/dev"
    - ssh -T ubuntu@$APP_HOST "chmod 777 ~/apps/dev"
    - scp docker-compose.yml ubuntu@$APP_HOST:~/apps/dev
    - scp .env ubuntu@$APP_HOST:~/apps/dev
    - ssh -T ubuntu@$APP_HOST "docker login -u $CI_DEPLOY_USER -p $CI_DEPLOY_PASSWORD $CI_REGISTRY && cd apps/dev && docker-compose pull && docker-compose up -d"
    - ssh -T ubuntu@$APP_HOST "cd apps/dev && docker-compose run --rm web rake db:create db:migrate db:seed && docker-compose up -d"

deploy-branches:
  stage: deploy
  except:
    - dev
    - master
  image: alpine:latest
  before_script:
    - hostname
    - pwd
    - whoami
    - apk update && apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
    - export RANDOM_PORT=$(shuf -i 82-89 -n 1)
    - export RANDOM_PORT_DB=$(shuf -i 5434-5439 -n 1)
    - export NGINX_PORT="${RANDOM_PORT}:80"
    - export DATABASE_PORT="${RANDOM_PORT_DB}:5432"
    - export APP_HOST_PORT="${APP_HOST}:${RANDOM_PORT}"

  script:
    - env | grep -E 'CI_REGISTRY_IMAGE|CI_COMMIT_SHORT_SHA|DB_|REDIS_URL|RACK_ENV|RAILS_|SECRET_KEY_BASE|SMTP_|GMAIL_|VUE_APP_BASE_HOST|APP_HOST|NGINX_PORT|DATABASE_PORT|APP_HOST_PORT' > .env
    - chmod 744 docker-compose.yml
    - chmod 744 .env
    - ssh -T ubuntu@$APP_HOST "mkdir -p ~/apps/\"$CI_COMMIT_REF_NAME/\""
    - ssh -T ubuntu@$APP_HOST "chmod 777 ~/apps/\"$CI_COMMIT_REF_NAME\""
    - scp docker-compose.yml ubuntu@$APP_HOST:~/apps/$CI_COMMIT_REF_NAME
    - scp .env ubuntu@$APP_HOST:~/apps/$CI_COMMIT_REF_NAME
    - ssh -T ubuntu@$APP_HOST "docker login -u $CI_DEPLOY_USER -p $CI_DEPLOY_PASSWORD $CI_REGISTRY && cd apps/$CI_COMMIT_REF_NAME && docker-compose pull && docker-compose up -d"
    - ssh -T ubuntu@$APP_HOST "cd apps/\"$CI_COMMIT_REF_NAME/\" && docker-compose run --rm web rake db:create db:migrate db:seed && docker-compose up -d"
```

This process has 3 stages:

First of all - *__building__*

This params setting by default

``` diff
docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_REGISTRY
```

This variables we set in `variables  Gitlab CI`

``` diff
 - env | grep -E 'CI_REGISTRY_IMAGE|CI_COMMIT_SHORT_SHA|DB_|REDIS_URL|RACK_ENV|RAILS_|SECRET_KEY_BASE|SMTP_|GMAIL_|VUE_APP_BASE_HOST|APP_HOST|NGINX_PORT|DATABASE_PORT|APP_HOST_PORT' > .env
```

Then docker-compose need to build and deploy to `CI_REGISTRY_IMAGE`

``` diff
docker-compose build
docker-compose push
```

The next stage is a *__deploy__*

in `before_scripts` adding OPEN SSH with key without confirmation.

``` diff
before_script:
    - hostname
    - pwd
    - whoami
    - apk update && apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
```

Here we're working with our server `(Amazon, Heroku, Corporative)`.

``` diff
scp docker-compose.yml ubuntu@$APP_HOST:~/apps
scp .env ubuntu@$APP_HOST:~/apps
```

Than all variables set in `.env` from *__Gitlab CI variables__*

`APP-HOST` - it's our variable from gitlab. We don't need to set.

``` diff
ssh -t ubuntu@$APP_HOST "docker login -u $CI_DEPLOY_USER -p $CI_DEPLOY_PASSWORD $CI_REGISTRY && cd apps && docker-compose pull && docker-compose up -d"
```

*~/apps* - this is our directory for application.

`CI_DEPLOY_USER $CI_DEPLOY_PASSWORD` comming from *__Deploying Tockens__*

``` diff
 only:
    - dev
```

+ only - it means, that autodeploy works only after pushing in *dev-branch*.


## Autodeploy on additional ports

*__How it works?__*

``` diff
deploy-branches:
  stage: deploy
  except:
    - dev
    - master
```

+ except - it means, that autodeploy works on all branches except *dev-branch*.

Our ports we can set automatically in *variable* like this

``` diff
variables:
    NGINX_PORT: "81:80"
    DATABASE_PORT: "5434:5432"
    APP_HOST_PORT: "${APP_HOST}:81"
```

But in autodeploy on additional ports we can set values from the interval:

``` diff
before_script:
    - export RANDOM_PORT=$(shuf -i 82-99 -n 1)
    - export RANDOM_PORT_DB=$(shuf -i 5434-5460 -n 1)
    - export NGINX_PORT="${RANDOM_PORT}:80"
    - export DATABASE_PORT="${RANDOM_PORT_DB}:5432"
    - export APP_HOST_PORT="${APP_HOST}:${RANDOM_PORT}"
```
 *__shuf__* - function for random value from interval.
 You can set interval up to 65000, in this case the probability of identical ports is minimal.

 *__All variables__* from before_script will be using in `.env` with help of this command:

``` diff
  - env | grep -E 'CI_REGISTRY_IMAGE|CI_COMMIT_SHORT_SHA|DB_|REDIS_URL|RACK_ENV|RAILS_|SECRET_KEY_BASE|SMTP_|GMAIL_|VUE_APP_BASE_HOST|APP_HOST|NGINX_PORT|DATABASE_PORT|APP_HOST_PORT' > .env
```

 Your `docker-compose.yml` looks like this:

 ``` diff
 version: '3'

services:
  db:
    image: postgres:12-alpine
    volumes:
      - /var/lib/postgresql
    ports:
      - "${DATABASE_PORT}"

  web:
    image: "${CI_REGISTRY_IMAGE-local}/app:${CI_COMMIT_SHORT_SHA-local}"
    build: .
    command: bash -c "cron && bundle exec rails s -p 3000 -b '0.0.0.0'"
    env_file:
      - .env
    links:
      - db
      - redis

  nginx:
    image: "${CI_REGISTRY_IMAGE-local}/nginx:${CI_COMMIT_SHORT_SHA-local}"
    build:
      context: .
      dockerfile: Dockerfile.nginx
    ports:
      - "${NGINX_PORT}"
    links:
      - web

  redis:
    image: redis:5.0-alpine
    command: redis-server --appendonly yes
    environment:
      - ALLOW_EMPTY_PASSWORD=yes
    volumes:
      - redis:/data

  sidekiq:
    image: "${CI_REGISTRY_IMAGE-local}/sidekiq:${CI_COMMIT_SHORT_SHA-local}"
    build: .
    environment:
      - REDIS_URL
    env_file:
       - .env
    links:
      - redis
    command: bash -c "bundle exec sidekiq -c 2"

volumes:
  redis:
  postgres:

 ```
