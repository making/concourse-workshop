## 初めてのパイプライン

### パイプラインの記述

`pipeline.yml`を作成して次の内容を記述してください。

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
          tag: bionic
      run:
        path: bash
        args: 
        - -c
        - |
          echo "Hello, world!"
```

### パイプラインの設定

`fly`コマンドでパイプラインを設定してください。

```
fly -t ws set-pipeline -p hello -c pipeline.yml
```

> `fly set-pipeline`は`fly sp`と省略することができます。

![image](https://user-images.githubusercontent.com/106908/73242647-e8ab9b00-41e8-11ea-8657-731f798d8766.png)

### パイプラインのpause解除

設定直後のパイプラインはpause状態になっており実行できません。

#### GUIで解除

Webブラウザでpauseを解除します。

ダッシュボードから`hello`パイプラインをクリックしてください。上部バーが青いのはpause状態であることを意味します。

![image](https://user-images.githubusercontent.com/106908/73242413-468bb300-41e8-11ea-94ca-807def15d698.png)

上部バーの▶️ボタンをクリックしてください。

![image](https://user-images.githubusercontent.com/106908/73242434-55726580-41e8-11ea-9038-4380207a4023.png)

上部バーが灰色になったらunpaseu状態になっています。

#### CUIで解除

`fly`コマンドでunpauseすることもできます。

```
fly -t ws unpause-pipeline -p hello
```

> `fly unpause-pipeline`は`fly up`と省略することができます。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/54333adf-cbb8-dfe3-db2a-c3652f714746.png)


### ジョブの実行

パイプラインがunpause状態になったらジョブを実行できます。`hello-world`ジョブをクリックしてください。

#### GUIで実行

Webブラウザ上で`hello-world`ジョブを実行します。右上の➕ボタンをクリックしてください。

![image](https://user-images.githubusercontent.com/106908/73242549-a1250f00-41e8-11ea-978b-23b2291a1132.png)

黄色になればジョブが開始された状態です。

![image](https://user-images.githubusercontent.com/106908/73242575-bac65680-41e8-11ea-9a10-f169cb9a82e4.png)

初回はDockerイメージのダウンロードが始まります。

![image](https://user-images.githubusercontent.com/106908/73242804-4809ab00-41e9-11ea-97c9-6c720ae6b526.png)

緑色になったらジョブが成功した状態です。

![image](https://user-images.githubusercontent.com/106908/73242833-5ce63e80-41e9-11ea-894d-e9da801cf324.png)

もう一度ジョブを実行すると(同じWorkerでジョブが実行された場合、)Dockerイメージのダウンロードは行われず、`"Hello, world!"`だけ出力されます。

![image](https://user-images.githubusercontent.com/106908/73242879-7be4d080-41e9-11ea-9672-1a147e7e47c8.png)

#### CUIで実行

`fly`コマンドでジョブをトリガーすることもできます。`--watch`をつけるとジョブの実行結果もターミナル上で確認できます。

```
fly -t ws trigger-job -j hello/hello-world --watch
```

> `fly set-pipeline`は`fly tj`と省略することができます。


![image](https://user-images.githubusercontent.com/106908/73242929-9d45bc80-41e9-11ea-8b2a-5b266ae85303.png)
