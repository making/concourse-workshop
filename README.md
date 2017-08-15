!!!!WIP!!!!

コンテンツの要望があればissueにあげてください

## 準備

```
fly -t ws login -n myteam https://concourse.example.com
```


## 初めてのパイプライン

### パイプラインの記述

`pipeline.yml`

``` yaml
jobs:
- name: hello-world
  plan:
  - task: say-hello
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: ubuntu
      run:
        path: bash
        args: 
        - -c
        - |
          echo "Hello, world!"
```

### パイプラインの設定

```
fly -t ws set-pipeline -p hello -c pipeline.yml
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/4033757c-1350-db71-48ea-921ea5f7a19f.png)


### パイプラインのpause解除


#### GUIで解除

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/0d95a374-c0c4-ada5-1b48-6b8224f9c1da.png)


![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/4fca0b6d-4035-749d-f799-b2be15e952d7.png)


![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/3d6f66b0-70ba-8281-f927-9b2825216b65.png)


![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/9419831e-ed9e-7123-2145-dcc1170db54e.png)


#### CUIで解除

```
fly -t ws unpause-pipeline -p hello
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/54333adf-cbb8-dfe3-db2a-c3652f714746.png)


### ジョブの実行


#### GUIで実行

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/87f9f1d7-9025-ab39-72e3-954784f7a247.png)


![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/19e2a9ee-a627-6aa0-0001-ebc7f82b1aa4.png)


#### CUIで実行

```
fly -t ws trigger-job -j hello/hello-world --watch
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/abbbe412-f651-4c09-604d-6507e0a2b10e.png)



## 初めてのJavaアプリケーションのテスト

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
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd repo
          mvn test
```

```
fly -t ws set-pipeline -p hello-servlet -c pipeline.yml
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/87addfb4-cfbe-58e2-c7d7-4f4259c4b8e1.png)



```
fly -t ws unpause-pipeline -p hello-servlet
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/08c16fcb-8dac-d5d3-2ee7-57b211ab5c2b.png)

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/c7accc89-f537-38cf-7d31-fc5666f93860.png)

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/b5b51b98-37f3-65f1-3687-1ba4a0bed2ac.png)


```
fly -t ws trigger-job -j hello-servlet/unit-test --watch
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/31562d3f-6f71-4fc3-533a-9cd81a3cd76a.png)



## キャッシュの導入

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
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/fcf90e41-32c9-01aa-9ce8-f84d0361581b.png)

```
fly -t ws trigger-job -j hello-servlet/unit-test --watch
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/c04a97a7-afd9-33c9-dc23-8ef84f846add.png)

```
fly -t ws trigger-job -j hello-servlet/unit-test --watch
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/b4d56c82-5f5e-d733-e441-ef6e1b929803.png)



## 初めてのJavaアプリケーションのデプロイ

```
cf push

scp
```

## パッケージマネージャNexusの導入 

```
aa
```

## semvarリソースでのバージョンの管理

```
a
```

## developブランチの管理

```
a
```

## あああ

```
a
```
