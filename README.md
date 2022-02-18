# CI templates repository

* [CI templates repository](#ci-templates-repository)
* [Common information](#common-information)
   * [Trunk based development flow](#TBD-flow)
   * [Git flow](#git-flow)
   * [Pipelines stages descriptions](#pipelines-stages-descriptions)
      * [Common CI stages](#common-ci-stages)
         * [Generate dynamic pipeline](#generate-dynamic-pipeline)
         * [Deploy](#deploy)
            * [Kubernetes cluster settings](#kubernetes-cluster-settings)
            * [Helm values files](#helm-values-files)
                * [Per Cluster](#per-cluster)
                * [Per Service](#per-service)
      * [Universal](#universal)
         * [External ci](#external-ci)
         * [Build](#build)
         * [Deploy](#deploy-1)
      * [GO](#go)
         * [Lint](#lint)
         * [Test](#test)
         * [Build](#build-1)
         * [Deploy](#deploy-2)
      * [Python](#python)
         * [Lint](#lint-1)
         * [Test](#test-1)
         * [Build](#build-2)
         * [Deploy](#deploy-3)
      * [Rust](#rust)
         * [Lint](#lint-2)
         * [Test](#test-2)
         * [Build](#build-3)
         * [Deploy](#deploy-4)
* [Developer guide](#developer-guide)
   * [Pipeline GO](#pipeline-go)
      * [Prepare .gitlab-ci.yml](#prepare-gitlab-ciyml)
      * [Prepare Helm values files](#prepare-helm-values-files)
      * [Prepare Dockerfile](#prepare-dockerfile)
      * [Prepare CI/CD project environments](#prepare-cicd-project-environments)
   * [Pipeline Python](#pipeline-python)
      * [Prepare .gitlab-ci.yml](#prepare-gitlab-ciyml-1)
      * [Prepare Helm values files](#prepare-helm-values-files-1)
      * [Prepare Dockerfile](#prepare-dockerfile-1)
      * [Prepare CI/CD project environments](#prepare-cicd-project-environments-1)
* [Admin guide](#admin-guide)
   * [All enviroments description](#all-enviroments-description)


# Common information
Проект представляет собой набор модулей CI/CD и собранных из них пайплайнов, модули могут использоваться разными пайплайнами и переопределяться в используемом проекте.

## TBD flow
Во всех пайплайнах реализована следующая схема (создавать теги запрещается на уровне проекта):
```
MR/commit to master:
   Lint -> Test -> Generate dynamic pipeline -> Build -> Deploy (DLDEVEL) [auto] -> Destroy (DLDEVEL) [manual]

branch: feature-*
  Lint -> Test -> Generate dynamic pipeline -> Build -> Deploy (DLDEVEL) [manual] -> Destroy (DLDEVEL) [manual]

protected branch: release-*
protected tag: ^v\d+\.\d+\.\d+$
   Lint -> Test -> Generate dynamic pipeline -> Build -> Deploy (STAGE) [auto]
                                                         ↳ deploy (PROD) [manual]
```

## Git flow
Во всех пайплайнах реализована следующая схема:
```
protected tag(master): ^v\d+\.\d+\.\d+$
   Lint -> Test -> Generate dynamic pipeline -> Build -> Deploy (PRD) manual

branch: release-*
   Lint -> Test -> Generate dynamic pipeline -> Build -> Deploy (STAGE) auto

branch: feature-*/dev
   Lint -> Test -> Generate dynamic pipeline -> Build -> Deploy (DEV) manual/auto -> Destroy (FEATURE-*/DEV) [manual]
```

По `dev` и `feature` веткам деплоятся отдельные окружения в кластер разработки. Все стенды можно посмотреть в проекте по пути **Operations -> Environments**, где будет встроенный функционал управления окружениями.

:warning: Время жизни окружения 1 неделя, после оно автоматически удалиться!

## Pipelines stages descriptions
### Common CI stages
Одинаково описанным для всех пайплайнов являются ***CI_STAGE***'s:

#### Generate dynamic pipeline

Одинаковый ***CI_STAGE*** для всех пайплайнов `CI TMPL`, необходим для генерации динамическх `child pipeline` иходя из входных данных в переменных:
```bash
variables:
...
  CI_TMPL_HELM_RELEASE_NAMES: "go-tmpl,go-tmpl-2"
  CI_TMPL_KUBE_CLUSTERS_DEV: "k8s.dldevel"
  CI_TMPL_KUBE_CLUSTERS_STAGE: "k8s.stage"
  CI_TMPL_KUBE_CLUSTERS_PROD: "k8s.datapro,k8s.dataline"
...
```

#### Deploy
Деплой производится в зависимости от схемы описанной в ***Git flow/TBD flow*** (ручной или автоматический режим)

Для деплоя используются менеджер `HHX` (Helm Helper eXecutor), менеджер имплементирует ограниченный набор методов пакетного менеджера `helm` + вспомогательные функции специфичные для `CI TMPL`:  
- [Страница проекта HHX](https://git.wildberries.ru/devops/ci/hhx)
- [Страница проекта helm](https://helm.sh/)

Деплой можно разделить на 3 основных этапа:
- Получение конфига для `kubernetes cluster`
- Получение результируещего `helm values`
- Деплой релиза

Переменные для кастомизации:
| Variables                                 | Description                              | Default                             |
| ------------------------------------------| -----------------------------------------| ------------------------------------|
| `CI_TMPL_HELM_CHART_NAME`                 | Имя Helm чарта.                          | [common-deploy](https://git.wildberries.ru/devops/helm/charts/common-deploy) |
| `CI_TMPL_HELM_CHART_VERSION`              | Версия Helm чарта                        | `2.0.15`                             |
| `CI_TMPL_HELM_REPO_NAME`                  | Имя Helm repository                      | `wb`                                |
| `CI_TMPL_HELM_REPO_ADDR`                  | Адрес Helm repository                    | `https://helm.wildberries.ru/`      |
| `CI_TMPL_HELM_RELEASE_VALUE_FILE_POSTFIX` | Постфикс имени values файлов             | `values.yml`                        |
| `CI_TMPL_HELM_RELEASE_TIMEOUT`            | Helm release timeout (sec)               | `600`                               |
| `CI_TMPL_HELM_RELEASE_WAIT`               | Включение/отключение Helm wait           | `true`                              |
| `CI_TMPL_HELM_NAMESPACE_CREATE`           | Создание/или нет kubernetes namespace    | `false`                             |
| `CI_TMPL_HELM_RELEASE_NAMES`              | Имя Kubernetes релизов/сервисов/приложений, через запятую. В **CI_JOB**'е для окружения доступна как: `$CI_TMPL_HELM_RELEASE_NAME` | `-`                                 |
| `CI_TMPL_KUBE_CLUSTERS_DEV`               | Имя Kubernetes кластеров для окружения `DEV`, через запятую. В **CI_JOB**'е для окружения доступна как: `$CI_TMPL_HELM_RELEASE_CLUSTER` | `-` |
| `CI_TMPL_KUBE_CLUSTERS_STAGE`             | Имя Kubernetes кластеров для окружения `STAGE`, через запятую. В **CI_JOB**'е для окружения доступна как: `$CI_TMPL_HELM_RELEASE_CLUSTER` | `-` |
| `CI_TMPL_KUBE_CLUSTERS_PROD`              | Имя Kubernetes кластеров для окружения `PROD`, через запятую. В **CI_JOB**'е для окружения доступна как: `$CI_TMPL_HELM_RELEASE_CLUSTER` | `-` |
| `CI_TMPL_VAULT_ADDR`                      | Vault address                            | `https://vault.wildberries.ru:8200` |
| `VAULT_JWT_ROLE_${CI_TMPL_HELM_RELEASE_CLUSTER}` | Имя Vault JWT Role, необходим для поучения kubeconfig для окружения, генерируется автоматически. Можно переопределить при крайней необходимости. В **CI_JOB**'е для окружения доступна как: `$CI_TMPL_VAULT_JWT_ROLE` | `dynamically` |
| `DOCKERFILE_DIR`                         | Директория для Dockerfiles (используется только при `Deploy type`: `Per Cluster`) | `./docker`                             |


##### Kubernetes cluster settings

HHX пытается получить `KubeConfig` из 2х источников:

- Vault
- System Environment

***Vault***

Для получения из `Vault` используются переменные:

1. `CI_TMPL_VAULT_ADDR`
2. `VAULT_JWT_ROLE_${CI_TMPL_HELM_RELEASE_CLUSTER}`

Переменная `VAULT_JWT_ROLE_${CI_TMPL_HELM_RELEASE_CLUSTER}` является не обязательной и используется только в случае когда для вашего кластера используется специфичная `VAULT JWT ROLE`.  
В **CI_JOB**'е для окружения доступна как: `$CI_TMPL_VAULT_JWT_ROLE`. В дефолтном поведении(определение перменной игнорируется) `CI_TMPL_VAULT_JWT_ROLE` генерируется следующим образом:
```bash
CI_TMPL_VAULT_JWT_ROLE=k8s_clusters-$CI_TMPL_HELM_RELEASE_CLUSTER-$(echo $CI_PROJECT_PATH |cut -d/ -f1)
```
:information_source: `VAULT JWT ROLE` создается при запросе на интеграцию с `Vault`.  
:information_source: `$(echo $CI_PROJECT_PATH |cut -d/ -f1)` - это имя основной группы вашего проекта.  

***System Environment***

Необходимые переменные в настройках CI в группе или проекте `gitlab`:

**cert/key access (есть доступ на весь кластер)**
```bash
CI_KUBE_CA_${CI_TMPL_HELM_RELEASE_CLUSTER} (base64)
CI_KUBE_CERT_${CI_TMPL_HELM_RELEASE_CLUSTER} (base64)
CI_KUBE_KEY_${CI_TMPL_HELM_RELEASE_CLUSTER} (base64)
CI_KUBE_SERVER_${CI_TMPL_HELM_RELEASE_CLUSTER} (адрес k8s api server)
```
или

**token access (доступ только в свой неймспейс)**
```bash
CI_KUBE_TOKEN_${CI_TMPL_HELM_RELEASE_CLUSTER}
CI_KUBE_SERVER_${CI_TMPL_HELM_RELEASE_CLUSTER} (адрес k8s api server)
```

Диаграма получения `KubeConfig` для HHX:

```plantuml
group Get KubeConfig
group KubeConfig for Cluster from Vault

    HHX -[#blue]> VAULT: Get KubeConfig for Cluster

    VAULT -[#green]> HHX: Return KubeConfig

else No KubeConfig for this Cluster

    group KubeConfig for Cluster Namespace from Vault
      HHX -[#blue]> VAULT : Get KubeConfig for Cluster Namespace
      VAULT -[#green]> HHX: Return KubeConfig
    else No KubeConfig for this Cluster Namespace
      VAULT -[#red]> HHX: No KubeConfig!
    end
end
else No KubeConfig for this Cluster from Vault

  group KubeConfig for Cluster from System Env

    HHX -[#blue]> "SYSTEM ENV" : Get Admin KubeConfig for Cluster
    "SYSTEM ENV" -[#green]> HHX: Return KubeConfig
    else No Admin KubeConfig for this Cluster

      group KubeConfig for Cluster by CERT ENV
        HHX -[#blue]> "SYSTEM ENV" : Get Admin KubeConfig for Cluster using CERT ENV
        "SYSTEM ENV" -[#green]> HHX: Return KubeConfig
      else No Admin KubeConfig for this Cluster using CERT ENV

        group KubeConfig for Cluster Namespace
          HHX -[#blue]> "SYSTEM ENV" : Get KubeConfig for Cluster Namespace using TOKEN ENV
          "SYSTEM ENV" -[#green]> HHX: Return KubeConfig
        else No KubeConfig for this Cluster Namespace using TOKEN ENV
          "SYSTEM ENV" -[#red]> HHX: Fail!!!
        end

      end

  end

end
```

##### Helm values files

Расположить helm values файлы можу 2я способами(2 типа иерерхий):
- **Per Cluster**
- **Per Service**

Для каждого типа свойственно следующее поведение для получения результирующего `helm values`, мерж values осуществляется от частного к общему.  
Прим:  

<img src="./images/hiera_merge.png"  width="70%" height="70%">

:information_source: В некторых случаях можно обойтись без описания `values` для конкреного service/cluster, а иногда и одним общим `values` :surfer:

Файлы `helm values` мержатся в один результирующий и обрабатываются как шаблон для переменных из среды окружения `HHX`.  
Переменные доступные в этой среде:

- полный список по конфиг файлу: https://git.wildberries.ru/devops/ci/hhx/-/blob/master/config.yml.bkp
- маппинг по переменным:
```bash
CI_TMPL_HELM_REPO_NAME -> {{ Helm.Repo.Name }}
CI_TMPL_HELM_REPO_ADDR -> {{ Helm.Repo.Addr }}
CI_TMPL_HELM_CHART_NAME -> {{ Helm.Chart.Name }}
CI_TMPL_HELM_CHART_VERSION -> {{ Helm.Chart.Version }}
CI_TMPL_HELM_RELEASE_NAME -> {{ Helm.Release.Name }}
CI_TMPL_HELM_RELEASE_IMAGE -> {{ Helm.Release.Image }}
CI_TMPL_HELM_RELEASE_TAG -> {{ Helm.Release.Tag }}
CI_TMPL_HELM_RELEASE_VALUE_FILE_POSTFIX -> {{ Helm.Release.ValueFilePostfix }}
CI_TMPL_HELM_RELEASE_DEPLOY_FOLDER_PATH -> {{ Helm.Release.DeployFolderPath }}
CI_TMPL_HELM_RELEASE_NAMESPACE -> {{ Helm.Release.Namespace }}
CI_TMPL_HELM_RELEASE_CLUSTER -> {{ Helm.Release.Cluster }}
CI_TMPL_HELM_RELEASE_TIMEOUT -> {{ Helm.Release.Timeout }}
CI_TMPL_HELM_RELEASE_WAIT -> {{ Helm.Release.Wait }}
CI_TMPL_HELM_RELEASE_NAMESPACE_CREATE -> {{ Helm.Release.NamespaceCreate }}
CI_TMPL_HELM_RELEASE_ENVIRONMENT -> {{ Helm.Release.Environment }}
CI_TMPL_VAULT_ADDR -> {{ Vault.Addr }}
CI_TMPL_VAULT_TOKEN -> {{ Vault.Token }}
CI_TMPL_VAULT_JWT_ROLE -> {{ Vault.JwtRole }}
```

Таким образом например задается образ и тег в `values` файлах:
```
image:
  repository: {{.Helm.Release.Image}}
  tag: {{.Helm.Release.Tag}}
```

###### Per Cluster
В данной конфигурации `helm values` файлы распологаются в диретории `deploy` относительно имени кластера.

Иерархия файлов:
```bash
deploy
├── common
│   └── SOMETHING-${CI_TMPL_HELM_RELEASE_VALUE_FILE_POSTFIX}
└── ${CI_TMPL_HELM_RELEASE_CLUSTER}
    ├── SOMETHING-${CI_TMPL_HELM_RELEASE_VALUE_FILE_POSTFIX}
    └── ${CI_TMPL_HELM_RELEASE_NAME}
        └── SOMETHING-${CI_TMPL_HELM_RELEASE_VALUE_FILE_POSTFIX}

${DOCKERFILE_DIR}
└── Dockerfile.${CI_TMPL_HELM_RELEASE_NAME}      
```

Пример для конфигурации `.gitlab-ci.yml`:
```bash
variables:
...
  CI_TMPL_HELM_RELEASE_NAMES: "go-tmpl,go-tmpl-2"
  CI_TMPL_HELM_RELEASE_NAMESPACE: devops
  # list of clusters for deploy
  CI_TMPL_KUBE_CLUSTERS_DEV: "k8s.dldevel"
  CI_TMPL_KUBE_CLUSTERS_STAGE: "k8s.stage"
  CI_TMPL_KUBE_CLUSTERS_PROD: "k8s.datapro,k8s.dataline"
...
```
Иерархия файлов будет следующей:
```bash
deploy
├── common
│   └── base-values.yml
├── k8s.dataline
│   ├── common-cluster-values.yml
│   ├── go-tmpl
│   │   ├── 01-values.yml
│   │   └── 02-values.yml
│   └── go-tmpl-2
│       ├── 01-values.yml
│       └── foo-values.yml
├── k8s.datapro
│   ├── common-cluster-values.yml
│   └── go-tmpl-2
│       ├── 01-values.yml
│       └── bar-values.yml
├── k8s.dldevel
│   ├── common-cluster-values.yml
│   └── go-tmpl-2
│       ├── foo-values.yml
│       └── bar-values.yml
└── k8s.stage
    ├── common-cluster-values.yml
    └── go-tmpl-2
        ├── yoo-values.yml
        └── 02-values.yml

docker
├── ${DOCKERFILE_COMMON_NAME}
├── Dockerfile.go-tmpl
└── Dockerfile.go-tmpl-2      
```

###### Per service
В данной конфигурации `helm values` файлы распологаются в диретории `deploy` относительно имени сервисов/приложений.

Иерархия файлов:
```bash
deploy
├── common
│   └── SOMETHING-${CI_TMPL_HELM_RELEASE_VALUE_FILE_POSTFIX}
└── ${CI_TMPL_HELM_RELEASE_NAME}
    ├── Dockerfile
    ├── SOMETHING-${CI_TMPL_HELM_RELEASE_VALUE_FILE_POSTFIX}
    └── ${CI_TMPL_HELM_RELEASE_CLUSTER}
        └── SOMETHING-${CI_TMPL_HELM_RELEASE_VALUE_FILE_POSTFIX}   
```

Пример для конфигурации `.gitlab-ci.yml`:
```bash
variables:
...
  CI_TMPL_HELM_RELEASE_NAMES: "go-tmpl,go-tmpl-2"
  CI_TMPL_HELM_RELEASE_NAMESPACE: devops
  # list of clusters for deploy
  CI_TMPL_KUBE_CLUSTERS_DEV: "k8s.dldevel"
  CI_TMPL_KUBE_CLUSTERS_STAGE: "k8s.stage"
  CI_TMPL_KUBE_CLUSTERS_PROD: "k8s.datapro,k8s.dataline"
...
```
Иерархия файлов будет следующей:
```bash
deploy
├── common
│   ├── ${DOCKERFILE_COMMON_NAME}
│   └── common-values.yml
├── go-tmpl
│   ├── Dockerfile
│   ├── common-svc-values.yml
│   ├── k8s.dataline
│   │   ├── 01-values.yml
│   │   └── 02-values.yml
│   ├── k8s.datapro
│   │   └── cho-values.yml
│   ├── k8s.dldevel
│   │   ├── 01-values.yml
│   │   └── 02-values.yml
│   └── k8s.stage
│       ├── foo-values.yml
│       └── bar-values.yml
└── go-tmpl-2
    ├── Dockerfile
    ├── common-svc-values.yml
    ├── k8s.dataline
    │   ├── 01-values.yml
    │   └── 02-values.yml
    ├── k8s.datapro
    │   └── cho-values.yml
    ├── k8s.dldevel
    │   ├── 01-values.yml
    │   └── 02-values.yml
    └── k8s.stage
        ├── foo-values.yml
        └── bar-values.yml
```

### Universal
Универсальный `Pipeline` подходит для проектов для которых не был реализован специальизированный `pipeline`, для проектов в которых `lint`, `test` джоб или они настолько специфичные, что шаблонизировать их нет технической возможности.

#### External ci
Есть возможность подключить CI из другого проекта, будет запущен перед джобами: `Build`, `Deploy`. Таким образом можно подключить специфичные вещи для вашого проекта извне. Например специфичные для вашего языка программирования линты и тесты и тп.

Переменные для настройки:
| Variables                                 | Description                                            | Default               |
| ------------------------------------------| -------------------------------------------------------| ----------------------|
| `CI_TMPL_EXTERNAL_CI_PROJECT`             | Проект (от корня gitlab, прим: devops/ci/external-ci) | `-`                   |
| `CI_TMPL_EXTERNAL_CI_PROJECT_REF`         | Ветка, тег и тп                                        | `-`                   |
| `CI_TMPL_EXTERNAL_CI_TARGET`              | Yaml файл CI для подключения. Прим: `/main-ci.yml`     | `-`                   |

:information_source: Все переменные должны быть определены, если есть потребность подключить `external ci`

#### Build
Необходимо самостоятельно описать Dockerfile's и разместить их в проекте.  
В зависимости от выбранного типа([Per Cluster](#per-cluster), [Per Service](#per-service)) иерархии каталогов CI TMPL, путь до `Dockerfile`'s будет отличаться.  
***Per Cluster***
```bash
${DOCKERFILE_DIR}
├── ${DOCKERFILE_COMMON_NAME}
└── Dockerfile.${CI_TMPL_HELM_RELEASE_NAME}      
```
***Per Service***
```bash
deploy
├── common/${DOCKERFILE_COMMON_NAME}
└── ${CI_TMPL_HELM_RELEASE_NAME}
    └── Dockerfile
```

:warning: в независимости от выбранного типа ([Per Cluster](#per-cluster), [Per Service](#per-service)) иерархии каталогов CI TMPL, для билда будет использоваться `Dockerfile` от общего к частному: если нет `Dockerfile` по пути для сервиса, будет использоваться общий в каталоге соответсвующий выбранному типу иерархии.

Файл необходимо подготовить самостоятельно.  

#### Deploy
См. [deploy](#deploy)

### GO
#### Lint
Используется `golangci-lint`, пакеты указываются в переменной `GO_PACKAGES`.  

Для более тонкой настройки можно поместить в корень проекта файл конфигурации `.golangci.yml`.

Пример:
```
---
run:
  concurency: 4
  deadline: 2m
  issues-exit-code: 1
  skip-files:
    - ".+_test.go"
    - "vendor/*"

output:
  format: tab
  print-issued-lines: true
  print-linter-name: true

linters:
  enable-all: true
  disable:
    - godox
    - lll
    - goerr113
    - funlen
  fast: false

issues:
  exclude-use-default: false
  max-issues-per-linter: 100
  max-same-issues: 4
  new: false
```

#### Test
Выполняется 2 теста:
- go test (`go test -mod=vendor -cover $GO_PACKAGES`)
- gocov test/report (`export GOFLAGS="-mod=vendor"; gocov test $GO_PACKAGES | gocov report`)


#### Build
В зависимости от выбранного типа([Per Cluster](#per-cluster), [Per Service](#per-service)) иерархии каталогов CI TMPL, путь до `Dockerfile`'s будет отличаться.  
***Per Cluster***
```bash
${DOCKERFILE_DIR}
├── ${DOCKERFILE_COMMON_NAME}
└── Dockerfile.${CI_TMPL_HELM_RELEASE_NAME}      
```
***Per Service***
```bash
deploy
├── common/${DOCKERFILE_COMMON_NAME}
└── ${CI_TMPL_HELM_RELEASE_NAME}
    └── Dockerfile
```

:warning: В независимости от выбранного типа ([Per Cluster](#per-cluster), [Per Service](#per-service)) иерархии каталогов CI TMPL, для билда будет использоваться `Dockerfile` от общего к частному - если нет `Dockerfile` по пути для сервиса, будет использоваться общий в катлоге соответсвующией выбронному типу иерархии.

Файл необходимо подготовить самостоятельно или взять из проекта `devops/ci/templates` по пути `./go/Dockerfile` - это общее описание сборки, подходит для простых приложений с одним бинарным файлом. В большинстве случаев этого достаточно.  

`./go/Dockerfile`:
```
ARG GO_VER
ARG ALPINE_VER
FROM git.wildberries.ru:4567/devops/docker/golang:${GO_VER}-alpine${ALPINE_VER} as builder
ARG GO_MAIN_PATH
ARG VERSION
WORKDIR /src
COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -mod=vendor -a -installsuffix cgo -o app -ldflags "-X 'main.version=${VERSION}'" ./${GO_MAIN_PATH}
FROM git.wildberries.ru:4567/devops/docker/alpine:${ALPINE_VER}
WORKDIR /root/
COPY --from=builder /src/app .
CMD ["./app"]
```
Для корректного использования зависимостей из внутренних приватных репозиториев существует два способа авторизации внутри контейнера:
- авторизация через ```git config``` с пробрасыванием токена:

  ```ARG CI_JOB_TOKEN```

  ```RUN git config --global --add url."https://gitlab-ci-token:${CI_JOB_TOKEN}@git.wildberries.ru/".insteadOf "https://git.wildberries.ru/"```

- авторизация через .netrc файл с записью токена в файл внутри контейнера:

  ```ARG CI_JOB_TOKEN```

  ```RUN printf "machine git.wildberries.ru\nlogin gitlab-ci-token\npassword ${CI_JOB_TOKEN}\n" > ~/.netrc```

В общем случае рекомендуется первый способ, второй требуется для подключения либ, где авторизация зашита через ```.netrc```

`CI_JOB_TOKEN` генерируется с правами пользователя создавшего pipeline, следовательно у пользователя выполняющего сборку должен быть доступ к репозиториям из списка зависимостей.

#### Deploy
См. [deploy](#deploy)


### Python
#### Lint
Используется `pylint`, версия линтера указываются в переменной `PYLINT_VERSION`. Линтер рекурсивно чекает файлы в списке каталогов с кодом `PY_LINT_PATHS`.

Для более тонкой настройки можно поместить в корень проекта файл конфигурации `.pylintrc`.

Пример:
```
[MASTER]
ignore=
    .venv,
    venv,
disable=
    C0114, # missing-module-docstring
    C0116, # Missing function or method docstring (missing-function-docstring)
    W0612, # Unused variable '???' (unused-variable)
```
Все PY окружение линтера кэшируется, список каталогов:
- .cache/pip
- .venv/

#### Test
Для запуска теста по умолчанию выбран `tox`. Используя файл конфигурации `tox.ini` в проекте, возможно более тонко настроить какой фреймворк для тестов использовать, следить за зависимостями и тд.
В [проекте шаблоне](https://git.wildberries.ru/devops/ci/projects/python-tmpl) для примера был настроен простой `pytest`.

Все PY окружение теста кэшируется, список каталогов:
- .cache/pip
- .venv/
- .tox/


#### Build
В зависимости от выбранного типа([Per Cluster](#per-cluster), [Per Service](#per-service)) иерархии каталогов CI TMPL, путь до `Dockerfile`'s будет отличаться.  
***Per Cluster***
```bash
${DOCKERFILE_DIR}
├── ${DOCKERFILE_COMMON_NAME}
└── Dockerfile.${CI_TMPL_HELM_RELEASE_NAME}      
```
***Per Service***
```bash
deploy
├── common/${DOCKERFILE_COMMON_NAME}
└── ${CI_TMPL_HELM_RELEASE_NAME}
    └── Dockerfile
```

:warning: В независимости от выбранного типа ([Per Cluster](#per-cluster), [Per Service](#per-service)) иерархии каталогов CI TMPL, для билда будет использоваться `Dockerfile` от общего к частному - если нет `Dockerfile` по пути для сервиса, будет использоваться общий в катлоге соответсвующией выбронному типу иерархии.

Файл необходимо подготовить самостоятельно или взять из проекта `devops/ci/templates` по пути `./python/Dockerfile`.  

`./python/Dockerfile`:
```
ARG PY_IMAGE
ARG PY_IMAGE_VERSION

# first stage
FROM ${PY_IMAGE}:${PY_IMAGE_VERSION} AS requirements-cache
COPY requirements.txt .

# install dependencies to the local user directory (eg. /root/.local)
RUN pip install --user -r requirements.txt

# second unnamed stage
FROM ${PY_IMAGE}:${PY_IMAGE_VERSION}
WORKDIR /app

# copy only the dependencies installation from the 1st stage image
COPY --from=requirements-cache /root/.local /root/.local
COPY . .

# update PATH environment variable
ENV PATH=/root/.local/bin:$PATH

CMD [ "python", "./app.py" ]
```

#### Deploy
См. [deploy](#deploy)

### Rust
#### Lint
Реализовано 2 джобы:
- ***lint*** (используется [rust-clippy](https://github.com/rust-lang/rust-clippy)). Линты можно настроить в файле TOML с именем clippy.toml или .clippy.toml.
- ***fmt*** (используется [rustfmt](https://github.com/rust-lang/rustfmt)) для более тонкой настройки используйте встроенный функционал разметки `#[rustfmt::skip::macros(target_macro_name)] or #[rustfmt::skip::attributes(target_attribute_name)]`.

#### Test
Для запуска теста релизована джоба: `unit tests` в рамках которой запускаются юнит тесты `cargo test`

#### Build

В зависимости от выбранного типа([Per Cluster](#per-cluster), [Per Service](#per-service)) иерархии каталогов CI TMPL, путь до `Dockerfile`'s будет отличаться.  
***Per Cluster***
```bash
${DOCKERFILE_DIR}
├── ${DOCKERFILE_COMMON_NAME}
└── Dockerfile.${CI_TMPL_HELM_RELEASE_NAME}      
```
***Per Service***
```bash
deploy
├── common/${DOCKERFILE_COMMON_NAME}
└── ${CI_TMPL_HELM_RELEASE_NAME}
    └── Dockerfile
```

:warning: В независимости от выбранного типа ([Per Cluster](#per-cluster), [Per Service](#per-service)) иерархии каталогов CI TMPL, для билда будет использоваться `Dockerfile` от общего к частному - если нет `Dockerfile` по пути для сервиса, будет использоваться общий в катлоге соответсвующией выбронному типу иерархии.

Файл необходимо подготовить самостоятельно.  

#### Deploy
См. [deploy](#deploy)


# Developer guide

Все реализованные пайплайны можно подключить на стадии создания проекта. Для этого были подготовлены проекты - `шаблоны`/`сэмплы`/`скелет` (как угодно), располагающиеся в группе https://git.wildberries.ru/devops/ci/projects.
К сожалению возможность использовать **Custom project templates** включена только в платную версию **Gitlab**, поэтому по этим проектам были получены `export` файлы для дальнейшего использования. Архивы экспорта хранятся в каталоге модулей.

Для создания проекта из `export` необходимо при содании проекта выбрать **Import project -> GitLab export** выбрать файл архива и заполнить необходимые данные.  

:warning: Для деплоя используется пакетный менеджер `HELM`, по этой причине рекомендуется все манипуляции с релизом (удаление, изменения) выполнять с помощью утилиты `helm`, это не касается операций обслуживания и дебага.
Если по какой то причине после ручного вмешательства в списке релизов:  
```
helm ls -n your_namespace
```
вы не наблюдаете ваш релиз, удалите ваш релиз и передеплойте кнопокой в `CI pipeline` снова:  
```
helm uninstall your_release -n your_namespace
```
Если при этом во время удаления релиз не найден, то удалите из секретов неймспейса все зависимости helm'а.


## Pipeline GO
Для подключения пайплайна необходимо выполнить пункты:
1. Prepare .gitlab-ci.yml
2. Prepare Helm values files
3. Prepare Dockerfile
4. Prepare CI/CD project environments

:exclamation: Если проект был создан из шаблона, по первым трем пунктам следует актуализировать данные в соотвествии с требованиями к проекту.

### Prepare .gitlab-ci.yml
Добавьте в `.gitlab-ci.yml` проекта описание:
```
.template_repo: &repo
  project: 'devops/ci/templates'
  ref: &ci_tmpl_vers 'v3.0.0'

variables:
  # hold pls
  CI_TMPL_PROJECT_VERSION: *ci_tmpl_vers

  GO_MAIN_PATH: ./cmd/app
  GO_VER: "1.14"
  GO_PACKAGES: ./cmd/...
  CI_TMPL_HELM_RELEASE_NAME: your-app
  CI_TMPL_HELM_RELEASE_NAMESPACE: your-app-ns
  HELM_CHART_NAME: wb/common-deploy
  HELM_CHART_VERSION: "1.0.11"

include:
  - {<<: *repo, file: /pipelines/go.yml}

```
для кастомизации GO пайплайна CI доступны переменные:  

| Variables           | Description                              | Default               |
| --------------------| -----------------------------------------| ----------------------|
| `CI_TMPL_HELM_RELEASE_NAME`      | Имя сервиса/приложения/релиза            | `-`                   |
| `CI_TMPL_HELM_RELEASE_NAMESPACE` | Имя Kubernetes namespace                 | `-`                   |
| `GO_MAIN_PATH`      | Путь до main.go для сборки               | `./app`               |
| `GO_IMAGE`          | Имя GO docker image                      | `harbor.wildberries.ru/docker-hub-proxy/library/golang` |
| `GO_VER`            | Версия GO docker image                   | `1.14`                |
| `GO_LINT_VER`       | Версия golangci-lint                     | `1.30`                |
| `GO_PACKAGES`       | Пути до пакетов в проекте (lint/test)    | `./...`               |
| `DOCKERFILE_DIR`    | Путь до Dockerfile, относительно проекта | `./Dockerfile`        |
| `HELM_CHART_NAME`   | Имя чарта для деплоя                     | `wb/common-deploy`    |
| `HELM_CHART_VERSION`| Версия чарта для деплоя                  | `1.0.11`               |

### Prepare Helm values files
Для деплоя необходимо убедиться в наличии `values` файлов в проекте. Подробнее можно ознакомиться в разделе [deploy](#deploy).

Содержимое файла может быть пустым, если дефолтные значения чарта ваc удовлетворяют (что навряд ли)

### Prepare Dockerfile
Подробно описано в [GO pipeline](#build)

:warning: В CI ожидается, что `Dockerfile` будет `multi stage`


### Prepare CI/CD project environments
Очень важный момент для деплоя в окружения - это наличие переменных с кредами в переменных самого проекта или группы.

Со списком переменных можно ознакомиться в разделе [deploy](#deploy)

:warning: Если переменные есть в группе в проекте, дублировать их не обязательно


## Pipeline Python
Для подключения пайплайна необходимо выполнить пункты:
1. Prepare .gitlab-ci.yml
2. Prepare Helm values files
3. Prepare Dockerfile
4. Prepare CI/CD project environments

:exclamation: Если проект был создан из шаблона, по первым трем пунктам нужно актуализировать данные в соотвествии с требованиями к проекту.

### Prepare .gitlab-ci.yml
Добавьте в `.gitlab-ci.yml` проекта описание:
```
.template_repo: &repo
  project: 'devops/ci/templates'
  ref: &ci_tmpl_vers 'v3.0.0'

variables:
  # hold pls
  CI_TMPL_PROJECT_VERSION: *ci_tmpl_vers

  PY_IMAGE: harbor.wildberries.ru/docker-hub-proxy/library/python
  PY_IMAGE_VERSION: 3.8-alpine
  PY_LINT_PATHS: ./app
  CI_TMPL_HELM_RELEASE_NAME: app
  CI_TMPL_HELM_RELEASE_NAMESPACE: python-tmpl
  HELM_CHART_NAME: wb/common-deploy
  HELM_CHART_VERSION: "1.0.11"

include:
  - {<<: *repo, file: /pipelines/python.yml}

```
для кастомизации Python пайплайна CI доступны переменные:  

| Variables               | Description                              | Default               |
| ------------------------| -----------------------------------------| ----------------------|
| `CI_TMPL_HELM_RELEASE_NAME`          | Имя сервиса/приложения/релиза            | `-`                   |
| `CI_TMPL_HELM_RELEASE_NAMESPACE`     | Имя Kubernetes namespace                 | `-`                   |
| `PY_IMAGE`              | Имя python docker image                  | `harbor.wildberries.ru/docker-hub-proxy/library/python` |
| `PY_IMAGE_VERSION`      | Версия python docker image               | `3.8-alpine`          |
| `PYLINT_VERSION`        | Версия pylint                            | `2.6.0`               |
| `PY_LINT_IMAGE`         | Имя python docker image для pylint       | `harbor.wildberries.ru/docker-hub-proxy/library/python` |
| `PY_LINT_IMAGE_VERSION` | Версия python docker image для pylint    | `3.8-alpine`          |
| `PY_LINT_PATHS`         | Список катлогоов и файлов, через пробел (юзается: lint) | `./app`               |
| `PY_TOX_VERSION`        | Версия tox                               | `3.20.1`              |
| `DOCKERFILE_DIR`       | Путь до Dockerfile, относительно проекта | `./Dockerfile`        |
| `HELM_CHART_NAME`       | Имя чарта для деплоя                     | `wb/common-deploy`    |
| `HELM_CHART_VERSION`    | Версия чарта для деплоя                  | `1.0.11`              |

### Prepare Helm values files
Для деплоя необходимо убедиться в наличии `values` файлов в проекте. Подробнее можно ознакомиться в разделе [deploy](#deploy).

Содержимое файла может быть пустым, если дефолтные значения чарта ваc удовлетворяют (что наврятли)

### Prepare Dockerfile
Подробно описано в [Python pipeline](#build-1)


### Prepare CI/CD project environments
Очень важный момент для деплоя в окружения - это наличие переменных с кредами в переменных самого проекта или группы.

Со списком переменных можно ознакомиться в разделе [deploy](#deploy)

:warning: Если переменные есть в группе в проекте, дублировать их не обязательно


# Admin guide

CI, описанный в проекте, реализован по принципу модульности, все доступные инструкции выделены логически и располагаются в иерархической структуре, с возможностью дальнейшего переиспользования в других модулях или пайплайнах.

## All enviroments description
Для настройки все параметры, которые могут влиять на поведения пайплайнов хранятся в переменных описанных в модулях:

| Variables               | Description                              | Default               |
| ------------------------| -----------------------------------------| ----------------------|
| `CI_TMPL_HELM_CHART_NAME`                 | Имя Helm чарта.                          | [common-deploy](https://git.wildberries.ru/devops/helm/charts/common-deploy) |
| `CI_TMPL_HELM_CHART_VERSION`              | Версия Helm чарта                        | `2.0.15`                             |
| `CI_TMPL_HELM_REPO_NAME`                  | Имя Helm repository                      | `wb`                                |
| `CI_TMPL_HELM_REPO_ADDR`                  | Адрес Helm repository                    | `https://helm.wildberries.ru/`      |
| `CI_TMPL_HELM_RELEASE_VALUE_FILE_POSTFIX` | Постфикс имени values файлов             | `values.yml`                        |
| `CI_TMPL_HELM_RELEASE_DEPLOY_FOLDER_PATH` | Путь до каталога с values файлами (per cluster или per service иерехией) | `./deploy` |
| `CI_TMPL_HELM_RELEASE_TIMEOUT`            | Helm release timeout (sec)               | `600`                               |
| `CI_TMPL_HELM_RELEASE_WAIT`               | Включение/отключение Helm wait           | `true`                              |
| `CI_TMPL_HELM_NAMESPACE_CREATE`           | Создание/или нет kubernetes namespace    | `false`                             |
| `CI_TMPL_HELM_RELEASE_NAMES`              | Имя Kubernetes релизов/сервисов/приложений, через запятую. В **CI_JOB**'е для окружения доступна как: `$CI_TMPL_HELM_RELEASE_NAME` | `-` |
| `CI_TMPL_KUBE_CLUSTERS_DEV`               | Имя Kubernetes кластеров для окружения `DEV`, через запятую. В **CI_JOB**'е для окружения доступна как: `$CI_TMPL_HELM_RELEASE_CLUSTER` | `-` |
| `CI_TMPL_KUBE_CLUSTERS_STAGE`             | Имя Kubernetes кластеров для окружения `STAGE`, через запятую. В **CI_JOB**'е для окружения доступна как: `$CI_TMPL_HELM_RELEASE_CLUSTER` | `-` |
| `CI_TMPL_KUBE_CLUSTERS_PROD`              | Имя Kubernetes кластеров для окружения `PROD`, через запятую. В **CI_JOB**'е для окружения доступна как: `$CI_TMPL_HELM_RELEASE_CLUSTER` | `-` |
| `CI_TMPL_VAULT_ADDR`                      | Vault address                            | `https://vault.wildberries.ru:8200` |
| `CI_TMPL_DEPLOYER_IMAGE`                  | Образ для деплоя хельмом                 | `harbor.wildberries.ru/devops/ci/hhx/hhx:v1-2-0-ce258cf2` |
| `VAULT_JWT_ROLE_${CI_TMPL_HELM_RELEASE_CLUSTER}` | Имя Vault JWT Role, необходим для поучения kubeconfig для окружения, генерируется автоматически. Можно переопределить при крайней необходимости. В **CI_JOB**'е для окружения доступна как: `$CI_TMPL_VAULT_JWT_ROLE` | `dynamically` |
| `DOCKERFILE_DIR`                         | Директория для Dockerfiles (используется только при `Deploy type`: `Per Cluster`) | `./docker`                             |
| `DOCKERFILE_COMMON_NAME`                 | Имя общего `Dockerfile`                  | `Dockerfile`          |
| `DOCKER_MIRROR`                          | Зеркало/прокси для DockerHub             | `docker-proxy.wildberries.ru` |
| `REGISTRY`                               | Реджестри для хранения собраных образов  | `harbor.wildberries.ru` |
| `REGISTRY_PROJECT`                       | Имя проекта в реджестри, по умолчанию равно имени корневой группы. | `$CI_PROJECT_ROOT_NAMESPACE` |
| `REGISTRY_IMAGE`                         | Имя содираемого образа | `$REGISTRY/$REGISTRY_PROJECT/$CI_PROJECT_PATH` |
| `VAULT_REGISTRY_JWT_ROLE`                | Vault JWT роль для доступа секрета для реджестри | `harbor-$REGISTRY_PROJECT-ro` |
| `VAULT_REGISTRY_SECRETS_PATH`            | Путь в vault до секрета с login и passsword для проекта в реджестри | `harbor/$REGISTRY_PROJECT/robot-accounts/$REGISTRY_PROJECT-rw` |
| `VAULT_REGISTRY_USER_SECRET_KEY`         | Имя vault secret key для получения пользователя | `user` |
| `VAULT_REGISTRY_PASSWORD_KEY`            | Имя vault secret key для получения пароля | `password` |
| `REGISTRY_USER`                          | Пользователь для доступа к реджестри. :warning: Не рекомендуется определять в .gitlab-ci.yml | :no_entry: |
| `REGISTRY_PASSWORD`                      | Пароль для доступа к реджестри. :warning: Не рекомендуется определять в .gitlab-ci.yml | :no_entry: |
| `GO_IMAGE`                               | Имя GO docker image                      | `harbor.wildberries.ru/docker-hub-proxy/library/golang` |
| `GO_VER`                                 | Версия GO docker image                   | `1.14`                |
| `GO_LINT_VER`                            | Версия golangci-lint                     | `1.30`                |
| `GO_PACKAGES`                            | Пути до пакетов в проекте (lint/test)    | `./...`               |
| `PY_IMAGE`                               | Имя python docker image                  | `harbor.wildberries.ru/docker-hub-proxy/library/python` |
| `PY_IMAGE_VERSION`                       | Версия python docker image               | `3.8-alpine`          |
| `PYLINT_VERSION`                         | Версия pylint                            | `2.6.0`               |
| `PY_LINT_IMAGE`                          | Имя python docker image для pylint       | `harbor.wildberries.ru/docker-hub-proxy/library/python` |
| `PY_LINT_IMAGE_VERSION`                  | Версия python docker image для pylint    | `3.8-alpine`          |
| `PY_LINT_PATHS`                          | Список катлогоов и файлов, через пробел (юзается: lint) | `./app`               |
| `PY_TOX_IMAGE`                           | Имя python docker image для tox          | `harbor.wildberries.ru/docker-hub-proxy/library/python` |
| `PY_TOX_VERSION`                         | Версия tox                               | `3.20.1`              |
| `PY_TOX_IMAGE_VERSION`                   | Версия python docker image для tox       | `3.8-alpine`          |
| `RUST_IMAGE`                             | Имя rust docker image                    | `harbor.wildberries.ru/docker-hub-proxy/library/rust` |
| `RUST_VER`                               | Версия rust docker image                 | `1.53.0`              |
