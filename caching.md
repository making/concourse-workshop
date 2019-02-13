## キャッシュの導入

### パイプラインの記述(更新)

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

### パイプラインの設定(更新)

```
fly -t ws set-pipeline -p hello-servlet -c pipeline.yml
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/fcf90e41-32c9-01aa-9ce8-f84d0361581b.png)

### ジョブの実行

```
fly -t ws trigger-job -j hello-servlet/unit-test --watch
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/c04a97a7-afd9-33c9-dc23-8ef84f846add.png)

```
fly -t ws trigger-job -j hello-servlet/unit-test --watch
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/b4d56c82-5f5e-d733-e441-ef6e1b929803.png)

