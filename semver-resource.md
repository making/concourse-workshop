## semverリソースでのバージョンの管理

### hello-servletプロジェクトのforkおよびhello-servlet-versionプロジェクトの作成

<img width="80%"  src="https://user-images.githubusercontent.com/106908/29493202-049509a8-85cd-11e7-8e18-8088ae8ae51b.png">

![image](https://user-images.githubusercontent.com/106908/29493212-24ba10d4-85cd-11e7-897e-f79119a6ed85.png)


![image](https://user-images.githubusercontent.com/106908/29493225-6283395e-85cd-11e7-8da0-1b7673de9d5e.png)


![image](https://user-images.githubusercontent.com/106908/29493227-7a2fad94-85cd-11e7-8907-12cc5febb20a.png)

![image](https://user-images.githubusercontent.com/106908/29493242-90828e72-85cd-11e7-9914-0aecf05d75f6.png)


![image](https://user-images.githubusercontent.com/106908/29493244-ad0578de-85cd-11e7-8bc5-353674a2c090.png)


![image](https://user-images.githubusercontent.com/106908/29493252-b6697358-85cd-11e7-9e1e-d706e7190664.png)



```
ssh-keygen -t rsa -f ~/.ssh/github
# Passphraseは空でOK
chmod 400 ~/.ssh/github
```

`~/.ssh/config`

```
host github.com
 HostName github.com
 IdentityFile ~/.ssh/github
 User git
```

![image](https://user-images.githubusercontent.com/106908/29495069-39131448-85f2-11e7-818f-2fcb521f9cec.png)

![image](https://user-images.githubusercontent.com/106908/29495074-4bc017d0-85f2-11e7-92d9-4db028bae144.png)

` ~/.ssh/github.pub`

![image](https://user-images.githubusercontent.com/106908/29495088-9dbbf09a-85f2-11e7-9306-f732b197ed78.png)

![image](https://user-images.githubusercontent.com/106908/29495085-8d025410-85f2-11e7-8f6b-b4b7e3383fa1.png)

![image](https://user-images.githubusercontent.com/106908/29495115-21826f44-85f3-11e7-871c-72e1c4d37291.png)

### リリースバージョンの設定

``` yaml
---
resources:
- name: repo
  type: git
  source:
    uri: git@github.com:pivot-tmaki/hello-servlet.git
    private_key: {{github-private-key}}
    branch: master
- name: repo-version
  type: semver
  source:
    uri: git@github.com:pivot-tmaki/hello-servlet-version.git
    branch: master
    private_key: {{github-private-key}}
    file: version
    driver: git
- name: cf
  type: cf
  source:
    api: {{cf-api}}
    username: {{cf-username}}
    password: {{cf-password}}
    organization: {{cf-org}}
    space: {{cf-space}}
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
- name: upload-to-nexus-snapshots
  plan:
  - get: repo
    passed:
    - unit-test
    trigger: true
  - task: mvn-deploy
    params: &NEXUS_SNAPSHOT
      NEXUS_URL: {{nexus-snapshot-url}}
      NEXUS_USERNAME: {{nexus-username}}
      NEXUS_PASSWORD: {{nexus-password}}
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
          cat > settings.xml <<EOF
          <settings>
            <servers>
              <server>
                 <id>repo</id>
                 <username>${NEXUS_USERNAME}</username>
                 <password>${NEXUS_PASSWORD}</password>
              </server>
            </servers>
          </settings>
          EOF
          mvn deploy -s settings.xml -DskipTests=true -DaltDeploymentRepository=repo::default::${NEXUS_URL} -Dmaven.wagon.http.ssl.insecure=true -D maven.wagon.http.ssl.ignore.validity.dates=true
- name: deploy
  plan:
  - get: repo
    passed:
    - upload-to-nexus-snapshots
    trigger: true
  - task: download-artifact
    params:
      <<: *NEXUS_SNAPSHOT
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
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd repo
          GROUP_ID=`grep '<groupId>' pom.xml | head -1  | sed -r 's/[ \f\n\r\t]+//g' | sed  -r 's|<.?groupId>||g'`
          ARTIFACT_ID=`grep '<artifactId>' pom.xml | head -1  | sed -r 's/[ \f\n\r\t]+//g' | sed  -r 's|<.?artifactId>||g'`
          VERSION=`grep '<version>' pom.xml | head -1  | sed -r 's/[ \f\n\r\t]+//g' | sed  -r 's|<.?version>||g'`
          URL=${NEXUS_URL}/`echo ${GROUP_ID} | sed 's|\.|/|g'`/${ARTIFACT_ID}/${VERSION}
          SNAPSHOT=`curl -k -s ${URL}/maven-metadata.xml | grep '<snapshotVersions>' -A 3 | grep 'value' | tr -d ' ' | tr -d '</value>'`
          echo "Download ${URL}/${ARTIFACT_ID}-${SNAPSHOT}.war"
          curl -k -u ${NEXUS_USERNAME}:${NEXUS_PASSWORD} -L -J -O ${URL}/${ARTIFACT_ID}-${SNAPSHOT}.war
          mv *.war ../build/ROOT.war
  - put: cf
    params:
      manifest: repo/manifest.yml
      path: build/ROOT.war
      current_app_name: hello-servlet
- name: tag-master
  plan:
  - aggregate:
    - get: repo
      passed:
      - deploy
    - get: repo-version
  - task: mvn-versions-set
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      - name: repo-version
      caches:
      - path: repo/m2   
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          VERSION=`cat repo-version/number`
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn versions:set -DnewVersion=${VERSION}
```

`credentials.yml`

``` yaml
---
cf-api: https://api.run.pivotal.io
cf-username: xxxxxx
cf-password: xxxxxx
cf-org: xxxxxx
cf-space: xxxxxx
nexus-snapshot-url: https://192.168.230.40/repository/maven-snapshots/
nexus-username: admin
nexus-password: admin123
github-private-key: |
  -----BEGIN RSA PRIVATE KEY-----
  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  -----END RSA PRIVATE KEY-----
```

```
fly -t ws set-pipeline -p hello-servlet -c pipeline.yml -l credentials.yml
```


![image](https://user-images.githubusercontent.com/106908/29495313-c308ff32-85f7-11e7-9c2d-2830fb99a4eb.png)

![image](https://user-images.githubusercontent.com/106908/29495408-a2fccfb4-85f9-11e7-9940-fff8d0aea08f.png)

```
fly -t ws trigger-job -j hello-servlet/tag-master --watch
```

![image](https://user-images.githubusercontent.com/106908/29495307-a3cd4402-85f7-11e7-98f0-fb4f8f2b0719.png)


### リリースとGitの更新

``` yaml
---
resources:
- name: repo
  type: git
  source:
    uri: git@github.com:pivot-tmaki/hello-servlet.git
    private_key: {{github-private-key}}
    branch: master
- name: repo-version
  type: semver
  source:
    uri: git@github.com:pivot-tmaki/hello-servlet-version.git
    branch: master
    private_key: {{github-private-key}}
    file: version
    driver: git
- name: cf
  type: cf
  source:
    api: {{cf-api}}
    username: {{cf-username}}
    password: {{cf-password}}
    organization: {{cf-org}}
    space: {{cf-space}}
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
- name: upload-to-nexus-snapshots
  plan:
  - get: repo
    passed:
    - unit-test
    trigger: true
  - task: mvn-deploy
    params: &NEXUS_SNAPSHOT
      NEXUS_URL: {{nexus-snapshot-url}}
      NEXUS_USERNAME: {{nexus-username}}
      NEXUS_PASSWORD: {{nexus-password}}
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
          cat > settings.xml <<EOF
          <settings>
            <servers>
              <server>
                 <id>repo</id>
                 <username>${NEXUS_USERNAME}</username>
                 <password>${NEXUS_PASSWORD}</password>
              </server>
            </servers>
          </settings>
          EOF
          mvn deploy -s settings.xml -DskipTests=true -DaltDeploymentRepository=repo::default::${NEXUS_URL} -Dmaven.wagon.http.ssl.insecure=true -D maven.wagon.http.ssl.ignore.validity.dates=true
- name: deploy
  plan:
  - get: repo
    passed:
    - upload-to-nexus-snapshots
    trigger: true
  - task: download-artifact
    params:
      <<: *NEXUS_SNAPSHOT
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
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd repo
          GROUP_ID=`grep '<groupId>' pom.xml | head -1  | sed -r 's/[ \f\n\r\t]+//g' | sed  -r 's|<.?groupId>||g'`
          ARTIFACT_ID=`grep '<artifactId>' pom.xml | head -1  | sed -r 's/[ \f\n\r\t]+//g' | sed  -r 's|<.?artifactId>||g'`
          VERSION=`grep '<version>' pom.xml | head -1  | sed -r 's/[ \f\n\r\t]+//g' | sed  -r 's|<.?version>||g'`
          URL=${NEXUS_URL}/`echo ${GROUP_ID} | sed 's|\.|/|g'`/${ARTIFACT_ID}/${VERSION}
          SNAPSHOT=`curl -k -s ${URL}/maven-metadata.xml | grep '<snapshotVersions>' -A 3 | grep 'value' | tr -d ' ' | tr -d '</value>'`
          echo "Download ${URL}/${ARTIFACT_ID}-${SNAPSHOT}.war"
          curl -k -u ${NEXUS_USERNAME}:${NEXUS_PASSWORD} -L -J -O ${URL}/${ARTIFACT_ID}-${SNAPSHOT}.war
          mv *.war ../build/ROOT.war
  - put: cf
    params:
      manifest: repo/manifest.yml
      path: build/ROOT.war
      current_app_name: hello-servlet
- name: tag-master
  plan:
  - aggregate:
    - get: repo
      passed:
      - deploy
    - get: repo-version
  - task: mvn-versions-set
    params:
      GIT_EMAIL: {{git-email}}
      GIT_NAME: {{git-name}}
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      - name: repo-version
      outputs:
      - name: output
      caches:
      - path: repo/m2   
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          VERSION=`cat repo-version/number`
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn versions:set -DnewVersion=${VERSION}
          rm -f pom.xml.versionsBackup
          shopt -s dotglob
          shopt -s extglob
          mv -f !(m2) ../output/
          cd ../output
          git config --global user.email "${GIT_EMAIL}"
          git config --global user.name "${GIT_NAME}"
          git add -A
          git commit -m "Release ${VERSION}"
  - put: repo
    params:
      repository: output
      tag: repo-version/number
```

`credentials.yml`

``` yaml
---
cf-api: https://api.run.pivotal.io
cf-username: xxxxxx
cf-password: xxxxxx
cf-org: xxxxxx
cf-space: xxxxxx
nexus-snapshot-url: https://192.168.230.40/repository/maven-snapshots/
nexus-username: admin
nexus-password: admin123
git-email: youremail@example.com
git-name: Your Name
github-private-key: |
  -----BEGIN RSA PRIVATE KEY-----
  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  -----END RSA PRIVATE KEY-----
```

![image](https://user-images.githubusercontent.com/106908/29495487-63cdd462-85fb-11e7-94b4-c473772a7d54.png)


### Nexusへのアップデートおよび、次の開発バージョン(パッチバージョン)へのアップデート

``` yaml
---
resources:
- name: repo
  type: git
  source:
    uri: git@github.com:pivot-tmaki/hello-servlet.git
    private_key: {{github-private-key}}
    branch: master
- name: repo-version
  type: semver
  source:
    uri: git@github.com:pivot-tmaki/hello-servlet-version.git
    branch: master
    private_key: {{github-private-key}}
    file: version
    driver: git
- name: cf
  type: cf
  source:
    api: {{cf-api}}
    username: {{cf-username}}
    password: {{cf-password}}
    organization: {{cf-org}}
    space: {{cf-space}}
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
- name: upload-to-nexus-snapshots
  plan:
  - get: repo
    passed:
    - unit-test
    trigger: true
  - task: mvn-deploy
    params: &NEXUS_SNAPSHOT
      NEXUS_URL: {{nexus-snapshot-url}}
      NEXUS_USERNAME: {{nexus-username}}
      NEXUS_PASSWORD: {{nexus-password}}
    config: &NEXUS_DEPLOY_CONFIG
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
          cat > settings.xml <<EOF
          <settings>
            <servers>
              <server>
                 <id>repo</id>
                 <username>${NEXUS_USERNAME}</username>
                 <password>${NEXUS_PASSWORD}</password>
              </server>
            </servers>
          </settings>
          EOF
          mvn deploy -s settings.xml -DskipTests=true -DaltDeploymentRepository=repo::default::${NEXUS_URL} -Dmaven.wagon.http.ssl.insecure=true -D maven.wagon.http.ssl.ignore.validity.dates=true
- name: deploy
  plan:
  - get: repo
    passed:
    - upload-to-nexus-snapshots
    trigger: true
  - task: download-artifact
    params:
      <<: *NEXUS_SNAPSHOT
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
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd repo
          GROUP_ID=`grep '<groupId>' pom.xml | head -1  | sed -r 's/[ \f\n\r\t]+//g' | sed  -r 's|<.?groupId>||g'`
          ARTIFACT_ID=`grep '<artifactId>' pom.xml | head -1  | sed -r 's/[ \f\n\r\t]+//g' | sed  -r 's|<.?artifactId>||g'`
          VERSION=`grep '<version>' pom.xml | head -1  | sed -r 's/[ \f\n\r\t]+//g' | sed  -r 's|<.?version>||g'`
          URL=${NEXUS_URL}/`echo ${GROUP_ID} | sed 's|\.|/|g'`/${ARTIFACT_ID}/${VERSION}
          SNAPSHOT=`curl -k -s ${URL}/maven-metadata.xml | grep '<snapshotVersions>' -A 3 | grep 'value' | tr -d ' ' | tr -d '</value>'`
          echo "Download ${URL}/${ARTIFACT_ID}-${SNAPSHOT}.war"
          curl -k -u ${NEXUS_USERNAME}:${NEXUS_PASSWORD} -L -J -O ${URL}/${ARTIFACT_ID}-${SNAPSHOT}.war
          mv *.war ../build/ROOT.war
  - put: cf
    params:
      manifest: repo/manifest.yml
      path: build/ROOT.war
      current_app_name: hello-servlet
- name: tag-master
  plan:
  - aggregate:
    - get: repo
      passed:
      - deploy
    - get: repo-version
  - task: mvn-versions-set
    params:
      GIT_EMAIL: {{git-email}}
      GIT_NAME: {{git-name}}
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      - name: repo-version
      outputs:
      - name: output
      caches:
      - path: repo/m2   
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          VERSION=`cat repo-version/number`
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn versions:set -DnewVersion=${VERSION}
          rm -f pom.xml.versionsBackup
          shopt -s dotglob
          shopt -s extglob
          mv -f !(m2) ../output/
          cd ../output
          git config --global user.email "${GIT_EMAIL}"
          git config --global user.name "${GIT_NAME}"
          git add -A
          git commit -m "Release ${VERSION}"
  - put: repo
    params:
      repository: output
      tag: repo-version/number
- name: upload-to-nexus-releases
  plan:
  - get: repo
    passed:
    - tag-master
    trigger: true
  - task: mvn-deploy
    params: &NEXUS_RELEASE
      NEXUS_URL: {{nexus-release-url}}
      NEXUS_USERNAME: {{nexus-username}}
      NEXUS_PASSWORD: {{nexus-password}}
    config:
      <<: *NEXUS_DEPLOY_CONFIG
- name: bump-to-next-patch-version
  plan:
  - aggregate:
    - get: repo
      passed:
      - upload-to-nexus-releases
      trigger: true
    - get: repo-version
      params:
        bump: patch
  - task: mvn-versions-set
    params:
      GIT_EMAIL: {{git-email}}
      GIT_NAME: {{git-name}}
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      - name: repo-version
      outputs:
      - name: output
      caches:
      - path: repo/m2   
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          VERSION=`cat repo-version/number`-SNAPSHOT
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn versions:set -DnewVersion=${VERSION} -DallowSnapshots
          rm -f pom.xml.versionsBackup
          shopt -s dotglob
          shopt -s extglob
          mv -f !(m2) ../output/
          cd ../output
          git config --global user.email "${GIT_EMAIL}"
          git config --global user.name "${GIT_NAME}"
          git add -A
          git commit -m "Bump to ${VERSION}"
  - put: repo
    params:
      repository: output
  - put: repo-version
    params:
      file: repo-version/number
```

`credentials.yml`

``` yaml
---
cf-api: https://api.run.pivotal.io
cf-username: xxxxxx
cf-password: xxxxxx
cf-org: xxxxxx
cf-space: xxxxxx
nexus-snapshot-url: https://192.168.230.40/repository/maven-snapshots
nexus-release-url: https://192.168.230.40/repository/maven-releases
nexus-username: admin
nexus-password: admin123
git-email: youremail@example.com
git-name: Your Name
github-private-key: |
  -----BEGIN RSA PRIVATE KEY-----
  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  -----END RSA PRIVATE KEY-----
```


![image](https://user-images.githubusercontent.com/106908/29495786-f5a06c24-8600-11e7-83f6-754e15e0df48.png)




![image](https://user-images.githubusercontent.com/106908/29495749-2448407a-8600-11e7-8952-78199f6956ef.png)
![image](https://user-images.githubusercontent.com/106908/29495756-584b2d24-8600-11e7-9681-7081d4c7fbec.png)


![image](https://user-images.githubusercontent.com/106908/29495728-e59fbde4-85ff-11e7-9877-6d146b8a3d67.png)

![image](https://user-images.githubusercontent.com/106908/29495739-fc228e66-85ff-11e7-8330-a70eb6d96dba.png)

### マイナーバージョン、メジャーバージョンへの手動アップデート

``` yaml
---
resources:
- name: repo
  type: git
  source:
    uri: git@github.com:pivot-tmaki/hello-servlet.git
    private_key: {{github-private-key}}
    branch: master
- name: repo-version
  type: semver
  source:
    uri: git@github.com:pivot-tmaki/hello-servlet-version.git
    branch: master
    private_key: {{github-private-key}}
    file: version
    driver: git
- name: cf
  type: cf
  source:
    api: {{cf-api}}
    username: {{cf-username}}
    password: {{cf-password}}
    organization: {{cf-org}}
    space: {{cf-space}}
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
- name: upload-to-nexus-snapshots
  plan:
  - get: repo
    passed:
    - unit-test
    trigger: true
  - task: mvn-deploy
    params: &NEXUS_SNAPSHOT
      NEXUS_URL: {{nexus-snapshot-url}}
      NEXUS_USERNAME: {{nexus-username}}
      NEXUS_PASSWORD: {{nexus-password}}
    config: &NEXUS_DEPLOY_CONFIG
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
          cat > settings.xml <<EOF
          <settings>
            <servers>
              <server>
                 <id>repo</id>
                 <username>${NEXUS_USERNAME}</username>
                 <password>${NEXUS_PASSWORD}</password>
              </server>
            </servers>
          </settings>
          EOF
          mvn deploy -s settings.xml -DskipTests=true -DaltDeploymentRepository=repo::default::${NEXUS_URL} -Dmaven.wagon.http.ssl.insecure=true -D maven.wagon.http.ssl.ignore.validity.dates=true
- name: deploy
  plan:
  - get: repo
    passed:
    - upload-to-nexus-snapshots
    trigger: true
  - task: download-artifact
    params:
      <<: *NEXUS_SNAPSHOT
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
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd repo
          GROUP_ID=`grep '<groupId>' pom.xml | head -1  | sed -r 's/[ \f\n\r\t]+//g' | sed  -r 's|<.?groupId>||g'`
          ARTIFACT_ID=`grep '<artifactId>' pom.xml | head -1  | sed -r 's/[ \f\n\r\t]+//g' | sed  -r 's|<.?artifactId>||g'`
          VERSION=`grep '<version>' pom.xml | head -1  | sed -r 's/[ \f\n\r\t]+//g' | sed  -r 's|<.?version>||g'`
          URL=${NEXUS_URL}/`echo ${GROUP_ID} | sed 's|\.|/|g'`/${ARTIFACT_ID}/${VERSION}
          SNAPSHOT=`curl -k -s ${URL}/maven-metadata.xml | grep '<snapshotVersions>' -A 3 | grep 'value' | tr -d ' ' | tr -d '</value>'`
          echo "Download ${URL}/${ARTIFACT_ID}-${SNAPSHOT}.war"
          curl -k -u ${NEXUS_USERNAME}:${NEXUS_PASSWORD} -L -J -O ${URL}/${ARTIFACT_ID}-${SNAPSHOT}.war
          mv *.war ../build/ROOT.war
  - put: cf
    params:
      manifest: repo/manifest.yml
      path: build/ROOT.war
      current_app_name: hello-servlet
- name: tag-master
  plan:
  - aggregate:
    - get: repo
      passed:
      - deploy
    - get: repo-version
  - task: mvn-versions-set
    params:
      GIT_EMAIL: {{git-email}}
      GIT_NAME: {{git-name}}
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      - name: repo-version
      outputs:
      - name: output
      caches:
      - path: repo/m2   
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          VERSION=`cat repo-version/number`
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn versions:set -DnewVersion=${VERSION}
          rm -f pom.xml.versionsBackup
          shopt -s dotglob
          shopt -s extglob
          mv -f !(m2) ../output/
          cd ../output
          git config --global user.email "${GIT_EMAIL}"
          git config --global user.name "${GIT_NAME}"
          git add -A
          git commit -m "Release ${VERSION}"
  - put: repo
    params:
      repository: output
      tag: repo-version/number
- name: upload-to-nexus-releases
  plan:
  - get: repo
    passed:
    - tag-master
    trigger: true
  - task: mvn-deploy
    params: &NEXUS_RELEASE
      NEXUS_URL: {{nexus-release-url}}
      NEXUS_USERNAME: {{nexus-username}}
      NEXUS_PASSWORD: {{nexus-password}}
    config:
      <<: *NEXUS_DEPLOY_CONFIG
- name: bump-to-next-patch-version
  plan:
  - aggregate:
    - get: repo
      passed:
      - upload-to-nexus-releases
      trigger: true
    - get: repo-version
      params:
        bump: patch
  - task: mvn-versions-set
    params:
      GIT_EMAIL: {{git-email}}
      GIT_NAME: {{git-name}}
    config: &MVN_VERSIONS_SET_CONFIG
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo
      - name: repo-version
      outputs:
      - name: output
      caches:
      - path: repo/m2   
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          VERSION=`cat repo-version/number`-SNAPSHOT
          cd repo
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn versions:set -DnewVersion=${VERSION} -DallowSnapshots
          rm -f pom.xml.versionsBackup
          shopt -s dotglob
          shopt -s extglob
          mv -f !(m2) ../output/
          cd ../output
          git config --global user.email "${GIT_EMAIL}"
          git config --global user.name "${GIT_NAME}"
          git add -A
          git commit -m "Bump to ${VERSION}"
  - put: repo
    params:
      repository: output
  - put: repo-version
    params:
      file: repo-version/number
- name: bump-to-next-minor-version
  plan:
  - aggregate:
    - get: repo
    - get: repo-version
      params:
        bump: minor
  - task: mvn-versions-set
    params:
      GIT_EMAIL: {{git-email}}
      GIT_NAME: {{git-name}}
    config:
      <<: *MVN_VERSIONS_SET_CONFIG
  - put: repo
    params:
      repository: output
  - put: repo-version
    params:
      file: repo-version/number
- name: bump-to-next-major-version
  plan:
  - aggregate:
    - get: repo
    - get: repo-version
      params:
        bump: major
  - task: mvn-versions-set
    params:
      GIT_EMAIL: {{git-email}}
      GIT_NAME: {{git-name}}
    config:
      <<: *MVN_VERSIONS_SET_CONFIG
  - put: repo
    params:
      repository: output
  - put: repo-version
    params:
      file: repo-version/number
```

![image](https://user-images.githubusercontent.com/106908/29495941-a8e800c4-8603-11e7-9534-132f4ea53f04.png)




