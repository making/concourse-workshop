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

```
test
```

## キャッシュの導入

```
.m2
```

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
