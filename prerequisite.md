## 事前準備

* [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
* [BOSH CLI](https://bosh.io/docs/cli-v2.html#install)

```
curl -L -J -O https://github.com/concourse/concourse/releases/download/v3.4.1/concourse-lite.yml
bosh create-env concourse-lite.yml
```

[参考:Install Log](https://gist.github.com/making/93a54be96722c06fa319be70c1d5bfbc)

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/8db7652f-0052-cb29-9869-d1e600743a85.png)

```
fly -t ws login -c http://192.168.100.4:8080
```
