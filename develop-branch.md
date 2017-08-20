## developブランチの管理

![image](https://user-images.githubusercontent.com/106908/29496111-edf7928a-8606-11e7-9a58-cc3545f16dd2.png)

``` yaml
---
resources:
- name: repo-develop
  type: git
  source:
    uri: git@github.com:pivot-tmaki/hello-servlet.git
    private_key: {{github-private-key}}
    branch: develop
- name: repo-master
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
- name: cf-develop
  type: cf
  source:
    api: {{cf-api}}
    username: {{cf-username}}
    password: {{cf-password}}
    organization: {{cf-org}}
    space: development
    skip_cert_check: true
- name: cf-master
  type: cf
  source:
    api: {{cf-api}}
    username: {{cf-username}}
    password: {{cf-password}}
    organization: {{cf-org}}
    space: production
    skip_cert_check: true
jobs:
- name: unit-test-develop
  plan:
  - get: repo
    resource: repo-develop
    trigger: true
  - task: mvn-test
    config: &MVN_TEST_CONFIG
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
    resource: repo-develop
    passed:
    - unit-test-develop
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
- name: deploy-develop
  plan:
  - get: repo
    resource: repo-develop
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
  - put: cf-develop
    params:
      manifest: repo/manifest.yml
      path: build/ROOT.war
      current_app_name: hello-servlet-develop
- name: merge-develop-to-master
  plan:
  - aggregate:
    - get: repo-src
      resource: repo-develop
      passed:
      - deploy-develop
    - get: repo-dest
      resource: repo-master
  - task: merge-develop-to-master
    params: &GIT_PARAMS
      GIT_EMAIL: {{git-email}}
      GIT_NAME: {{git-name}}
      SRC_BRANCH: develop
    config: &MERGE_SRC_TO_DEST
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: repo-src
      - name: repo-dest
      outputs:
      - name: merged 
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          shopt -s dotglob
          mv -f repo-dest/* merged/
          cd merged
          git config --global user.email "${GIT_EMAIL}"
          git config --global user.name "${GIT_NAME}"
          git remote add -f src ../repo-src
          git merge --no-edit src/${SRC_BRANCH}
  - put: repo-master
    params:
      repository: merged
- name: unit-test-master
  plan:
  - get: repo
    resource: repo-master
    trigger: true
    passed:
    - merge-develop-to-master
  - task: mvn-test
    config:
      <<: *MVN_TEST_CONFIG
- name: tag-master
  plan:
  - aggregate:
    - get: repo
      resource: repo-master
      trigger: true
      passed:
      - unit-test-master
    - get: repo-version
  - task: mvn-versions-set
    params:
      <<: *GIT_PARAMS
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
  - put: repo-master
    params:
      repository: output
      tag: repo-version/number
- name: upload-to-nexus-releases
  plan:
  - get: repo
    resource: repo-master
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
- name: deploy-master
  plan:
  - get: repo
    resource: repo-master
    passed:
    - upload-to-nexus-releases
    trigger: true
  - task: download-artifact
    params:
      <<: *NEXUS_RELEASE
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
          echo "Download ${URL}/${ARTIFACT_ID}-${VERSION}.war"
          curl -k -u ${NEXUS_USERNAME}:${NEXUS_PASSWORD} -L -J -O ${URL}/${ARTIFACT_ID}-${VERSION}.war
          mv *.war ../build/ROOT.war
  - put: cf-master
    params:
      manifest: repo/manifest.yml
      path: build/ROOT.war
      current_app_name: hello-servlet-master 
- name: bump-to-next-patch-version
  plan:
  - aggregate:
    - get: repo-src
      resource: repo-master
      passed:
      - deploy-master
      trigger: true
    - get: repo-dest
      resource: repo-develop
    - get: repo-version
      params:
        bump: patch
  - task: merge-master-to-develop
    params:
      <<: *GIT_PARAMS
      SRC_BRANCH: master
    config:
      <<: *MERGE_SRC_TO_DEST
  - task: just-move
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: merged
      outputs:
      - name: repo
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          shopt -s dotglob
          cp -r merged/* repo/
  - task: mvn-versions-set
    params:
      <<: *GIT_PARAMS
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
  - put: repo-develop
    params:
      repository: output
  - put: repo-version
    params:
      file: repo-version/number
- name: bump-to-next-minor-version
  plan:
  - aggregate:
    - get: repo
      resource: repo-develop
    - get: repo-version
      params:
        bump: minor
  - task: mvn-versions-set
    params:
      <<: *GIT_PARAMS
    config:
      <<: *MVN_VERSIONS_SET_CONFIG
  - put: repo-develop
    params:
      repository: output
  - put: repo-version
    params:
      file: repo-version/number
- name: bump-to-next-major-version
  plan:
  - aggregate:
    - get: repo
      resource: repo-develop
    - get: repo-version
      params:
        bump: major
  - task: mvn-versions-set
    params:
      <<: *GIT_PARAMS
    config:
      <<: *MVN_VERSIONS_SET_CONFIG
  - put: repo-develop
    params:
      repository: output
  - put: repo-version
    params:
      file: repo-version/number
```

![image](https://user-images.githubusercontent.com/106908/29496945-ecd53dc8-8618-11e7-8329-e2947aa23f47.png)
