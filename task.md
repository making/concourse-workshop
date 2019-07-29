## Taskのファイル分割


`booklit.yml`

```yaml
- name: booklit
  type: git
  source:
    uri: https://github.com/vito/booklit

jobs:
- name: unit
  plan:
  - get: booklit
    trigger: true
  - task: test
    file: booklit/ci/test.yml
```


```
fly -t ws set-pipeline -p booklit -c booklit.yml
fly -t ws unpause-pipeline -p booklit
```
