## 初めてのJavaアプリケーションのテスト

### パイプラインの記述

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

### パイプラインの設定

```
fly -t ws set-pipeline -p hello-servlet -c pipeline.yml
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/87addfb4-cfbe-58e2-c7d7-4f4259c4b8e1.png)

### パイプラインのpause解除


```
fly -t ws unpause-pipeline -p hello-servlet
```


![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/08c16fcb-8dac-d5d3-2ee7-57b211ab5c2b.png)

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/c7accc89-f537-38cf-7d31-fc5666f93860.png)

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/b5b51b98-37f3-65f1-3687-1ba4a0bed2ac.png)


### ジョブの再実行

```
fly -t ws trigger-job -j hello-servlet/unit-test --watch
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/31562d3f-6f71-4fc3-533a-9cd81a3cd76a.png)
