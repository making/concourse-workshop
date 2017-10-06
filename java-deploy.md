## Javaアプリケーションのデプロイ

### ジョブの連携

``` yaml
---
resources:
- name: repo
  type: git
  source:
    uri: https://github.com/making/hello-servlet.git

jobs:
- name: unit-test
  plan:
  - get: repo
    trigger: true
  - task: mvn-test
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      caches:
      - path: repo/m2   
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn test
- name: build-and-deploy
  plan:
  - get: repo
    passed:
    - unit-test
    trigger: true
  - task: mvn-package
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      caches:
      - path: repo/m2
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn package -DskipTests=true
          find target
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/cdb6a278-b70a-53e1-61bc-c3e10dadadea.png)


### Cloud Foundryへのデプロイ

``` yaml
---
resources:
- name: repo
  type: git
  source:
    uri: https://github.com/making/hello-servlet.git
- name: cf
  type: cf
  source:
    api: ((cf-api))
    username: ((cf-username))
    password: ((cf-password))
    organization: ((cf-org))
    space: ((cf-space))
    skip_cert_check: true
jobs:
- name: unit-test
  plan:
  - get: repo
    trigger: true
  - task: mvn-test
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      caches:
      - path: repo/m2   
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn test
- name: build-and-deploy
  plan:
  - get: repo
    passed:
    - unit-test
    trigger: true
  - task: mvn-package
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      outputs:
      - name: build
      caches:
      - path: repo/m2
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn package -DskipTests=true
          mv target/ROOT.war ../build
  - put: cf
    params:
      manifest: repo/manifest.yml
      path: build/ROOT.war
```

`credentials.yml`

``` yaml
---
cf-api: https://api.run.pivotal.io
cf-username: xxxxxx
cf-password: xxxxxx
cf-org: xxxxxx
cf-space: xxxxxx
```

```
fly -t ws set-pipeline -p hello-servlet -c pipeline.yml -l credentials.yml
```


![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/69aec1cd-155b-5e67-9e22-c418cfa78582.png)

```
fly -t ws trigger-job -j hello-servlet/build-and-deploy --watch
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/e7814494-b3fe-eafa-97aa-0f0cedcc3b71.png)


```
$ curl https://hello-servlet.cfapps.io

██╗     █████╗ ███╗   ███╗    ███████╗████████╗██╗██╗     ██╗          █████╗ ████████╗    ██╗    ██╗ █████╗ ██████╗ 
██║    ██╔══██╗████╗ ████║    ██╔════╝╚══██╔══╝██║██║     ██║         ██╔══██╗╚══██╔══╝    ██║    ██║██╔══██╗██╔══██╗
██║    ███████║██╔████╔██║    ███████╗   ██║   ██║██║     ██║         ███████║   ██║       ██║ █╗ ██║███████║██████╔╝
██║    ██╔══██║██║╚██╔╝██║    ╚════██║   ██║   ██║██║     ██║         ██╔══██║   ██║       ██║███╗██║██╔══██║██╔══██╗
██║    ██║  ██║██║ ╚═╝ ██║    ███████║   ██║   ██║███████╗███████╗    ██║  ██║   ██║       ╚███╔███╔╝██║  ██║██║  ██║
╚═╝    ╚═╝  ╚═╝╚═╝     ╚═╝    ╚══════╝   ╚═╝   ╚═╝╚══════╝╚══════╝    ╚═╝  ╚═╝   ╚═╝        ╚══╝╚══╝ ╚═╝  ╚═╝╚═╝  ╚═╝
```

### Cloud Foundryへのゼロダウンタイムアップデート


``` yaml
---
resources:
- name: repo
  type: git
  source:
    uri: https://github.com/making/hello-servlet.git
- name: cf
  type: cf
  source:
    api: ((cf-api))
    username: ((cf-username))
    password: ((cf-password))
    organization: ((cf-org))
    space: ((cf-space))
    skip_cert_check: true
jobs:
- name: unit-test
  plan:
  - get: repo
    trigger: true
  - task: mvn-test
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      caches:
      - path: repo/m2   
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn test
- name: build-and-deploy
  plan:
  - get: repo
    passed:
    - unit-test
    trigger: true
  - task: mvn-package
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      outputs:
      - name: build
      caches:
      - path: repo/m2
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn package -DskipTests=true
          mv target/ROOT.war ../build
  - put: cf
    params:
      manifest: repo/manifest.yml
      path: build/ROOT.war
      current_app_name: hello-servlet
```

### (おまけ) SCPで別サーバーへデプロイ

``` yaml
---
resources:
- name: repo
  type: git
  source:
    uri: https://github.com/making/hello-servlet.git
jobs:
- name: unit-test
  plan:
  - get: repo
    trigger: true
  - task: mvn-test
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      caches:
      - path: repo/m2   
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn test
- name: build-and-deploy
  plan:
  - get: repo
    passed:
    - unit-test
    trigger: true
  - task: mvn-package
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      outputs:
      - name: build
      caches:
      - path: repo/m2
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn package -DskipTests=true
          mv target/ROOT.war ../build/
  - task: scp
    params:
      SSH_USER: ((ssh-user))
      SSH_HOST: ((ssh-host))
      SSH_PRIVATE_KEY: ((ssh-private-key))
      SSH_BASE_DIR: ((ssh-base-dir))
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: ecsteam/scp-resource
      inputs:
      - name: build
      run:
        path: bash
        args: 
        - -c
        - |
          set -e
          cat > private_key <<EOF
          ${SSH_PRIVATE_KEY}
          EOF
          chmod 400 private_key
          scp -i private_key -o StrictHostKeyChecking=no build/* ${SSH_USER}@${SSH_HOST}:${SSH_BASE_DIR}/
```

`credentials.yml`

``` yaml
ssh-host: xxxxxx
ssh-user: xxxxxx
ssh-private-key: |
  -----BEGIN RSA PRIVATE KEY-----
  xxxxxx
  xxxxxx
  -----END RSA PRIVATE KEY-----
ssh-base-dir: /opt/tomcat/
```


```
fly -t ws set-pipeline -p hello-servlet -c pipeline.yml -l credentials.yml
```
