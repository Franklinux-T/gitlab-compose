
## Requirements

Create certificates for "*.example.com" in /srv/gitlab/ssl

- https://github.com/FiloSottile/mkcert
- openssl
- let's encrypt

## HOW-TO

Wake up, gitlab!

```shell
docker-compose up -d

echo "127.0.0.1 gitlab.example.com traefik.example.com registry.gitlab.example.com" | sudo tee -a /etc/hosts
```

- Log into gitlab.example.com , go to user settings:
    - Add ssh key to profile
    - Create 'personal access token'
    - Get gitlab-runner token api in CI/CD settings

### Gitlab-runner
https://docs.gitlab.com/runner/configuration/advanced-configuration.html

- Register the gitlab-runner: 

  * [WIP] Gitlab-runner example config:

   ```toml
   concurrent = 1
   check_interval = 0
  
   [session_server]
     session_timeout = 1800
  
   [[runners]]
     name = "a45488195e3c"
     url = "http://gitlab:80"
     token = "_GITLAB_RUNNER_TOKEN_"
     executor = "docker"
     clone_url = "http://gitlab:80"
     environment = ["DOCKER_DRIVER=overlay2"]
     [runners.custom_build_dir]
     [runners.docker]
       tls_verify = false
       image = "docker:latest"
       privileged = true
       disable_entrypoint_overwrite = false
       oom_kill_disable = false
       disable_cache = false
       cache_dir = "/cache"
       volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/    cache"]
       network_mode = "gitlabcompose_gitlab"
       shm_size = 0
     [runners.cache]
       Insecure = false
       [runners.cache.s3]
       [runners.cache.gcs]
   ```

  * Update token api

  ```shell
  docker exec gitlabrunner /bin/bash -c "export    GITLAB_RUNNER_TOKEN=XXXXXXXXXXXXXXXXXXXX; sed -i 's/   _GITLAB_RUNNER_TOKEN_/${GITLAB_RUNNER_TOKEN}/g' /etcgitlab-runner/   config.toml"
  ```



### Gitlab Registry

- Check gitlab-registry login

    ```shell
    DOCKER_USERNAME=username # gitlab credentials
    DOCKER_PASSWORD=password

    docker login -u=${DOCKER_USERNAME} -p=${DOCKER_PASSWORD}    registry.gitlab.example.com
    ```

### Snippets

- Traefik example config

    ```toml
    defaultEntryPoints = ["https","http"]

    [entryPoints]
      [entryPoints.dashboard]
        address = ":8080"
      [entryPoints.http]
        address = ":80"
        [entryPoints.http.redirect]
          entryPoint = "https"
      [entryPoints.https]
        address = ":443"
      [entryPoints.https.tls]
        [[entryPoints.https.tls.certificates]]
          certFile = "/ssl/example.com.pem"
          keyFile = "/ssl/example.com.key"

    [api]
    entrypoint="dashboard"
    #debug = true

    [docker]
    domain = "example.com"
    watch = true
    network = "gitlab"
    exposedbydefault = false
    ```

- Gitlab-ci.yml example

    ```yml
    stages:
      - build
      - test

    build:
      image: docker:stable
      services:
        - name:  docker:dind
      stage: build
      script:
        #- export # show environment variables
        - apk add --no-cache git
        - if [ -f "./faas-cli" ] ; then cp ./faas-cli /usr/local/bin/   faas-cli || 0 ; fi
        - if [ ! -f "/usr/local/bin/faas-cli" ] ; then apk add --no-cache     curl git && curl -sSL cli.openfaas.com | sh && chmod +x /usr/   local/bin/faas-cli && cp /usr/local/bin/faas-cli ./faas-cli ; fi
        # Build Docker image
        - /usr/local/bin/faas-cli build -f criptowatch-ohlc.yml     --build-arg ADDITIONAL_PACKAGE='make automake gcc musl-dev g++    python3-dev'
        - docker tag kammin/criptowatch-ohlc:latest     registry.gitlab.example.com/functions/criptowatch-ohlc:latest 
        - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD    registry.gitlab.example.com
        - docker push registry.gitlab.example.com/functions/    criptowatch-ohlc:latest
    #  after_script:
    #    - 
      only:
        - master

    test:
      image: docker:stable
      services:
        - docker:dind
      stage: test
      dependencies:
        - build
      script:
        - ""
      after_script:
        - ""
      only:
        - master
    ```