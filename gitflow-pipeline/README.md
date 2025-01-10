# Gitflow pipeline RU #

Спроектировать грамотный и понятный любому участнику процесса разработки CI/CD pipeline с учетом разных требований - еще та головоломка, как для рядовых developers, на которых иногда такие задачи сваливаются, так и опытных DevOps инженеров.

В данной статье демонстрируется подход по организации CI/CD pipeline c учетом таких требований как:

- реализация под [Gitlab CI/CD pipeline](https://docs.gitlab.com/ee/ci/pipelines/).
- процесс разработки проходит по правилам  [Gitflow workflow](./gitflow-worklow.md)
- необходимость выполнения действий в несколько этапов и для нескольких сред: DEV, QA or TEST, PROD.
- возможность указания режима запуска JOBs в pipeline: автоматически или вручную.

Описываемый далее CI/CD pipeline был специально спроектирован для Gitflow workflow поэтому был назван **Gitflow pipeline**.

Подход оказался достаточно удачным для организации  Gitlab CI/CD pipelines разного рода и уже не раз был применен в других проектах и доказал что являться гибким в организации и реорганизации pipeline, простым в настройке, и интуитивно-понятным для всех.

В общем виде **Gitflow pipeline** может выглядеть как:

```yaml
include: gitflow-pipeline-conditions.yml

variables:
  FEATURE_KEYS: "|build|test|deploy-d-manual|"
  BUGFIX_KEYS: "|build|test-skip|deploy-d-manual|deploy-q-manual|"
  MR_KEYS: "|build|test|mr-rules-manual|"
  DEVELOP_KEYS: "|build|test|deploy-d|deploy-q|"
  RELEASE_KEYS: "|build|test|deploy-p-manual|"
  HOTFIX_KEYS: "|build|test-skip|deploy-p-manual|"
```

где:

- FEATURE_KEYS, BUGFIX_KEYS, MR_KEYS, ... - это **CONDITIONS** при которых pipeline запускается, в данном случае при  PUSH в соответствующую ветку согласно Gitflow (см. [Conditions definition](#conditions-definition)).
- |build|test|deploy-d-manual| - это последовательность |KEY-1|KEY-2|KEY-N|  где за каждым  |KEY-X| закрепляются JOBs, которые будут запущены автоматически или вручную при запуске pipeline,  в данном случае при PUSH в соответствующую ветку. (см. [Job definition](#jobs-definition)).
- имена |KEY-1|KEY-2|KEY-N| заданы согласно naming convention. (см. [Keys naming convention](#keys-naming-convention)).

## Sample ##

Разберем пример: ```FEATURE_KEYS: "|build|test|deploy-d-manual|"``` - при PUSH в ветку "feature/*", автоматически запуститься pipeline и JOB закрепленный за key |build| и при успешном завершении job |build| далее автоматически запуститься job закрепленный за key |test|, далее при успешном завершении job |test|, активируется job закрепленный за key |deploy-d-manual|, но будет ожидать ручного запуска.

JOBs  в общем виде задаются как (пример для |build|):

```yaml
.job-build:
  stage: build
  script:
      - echo "execute script to build"
  variables:
    REGEX_KEY_AUTO: /\|(build|build-auto)\|/i
    REGEX_KEY_MANUAL: /\|build-manual\|/i
```

Закрепляем JOB за KEYs |build| или |build-auto| или |build-manual| -  задаем как regex в variables `REGEX_KEY_AUTO` и `REGEX_KEY_MANUAL`.

JOB определяем как [HIDDEN (Start the job name with a dot (.))](https://docs.gitlab.com/ee/ci/jobs/#hide-jobs).

Далее определяем два VISIBLE JOBs для ручного и автоматического запуска как:

```yaml
build:manual:
  extends:
    - .start-manual
    - .job-build

build:auto:
  extends:
    - .start-auto
    - .job-build
```

И теперь если для {BRANCH-NAME}_KEYS будут указаны KEYs |build| или |build-auto| - закрепленный JOB запуститься автоматически, а если |build-manual|  - закрепленный job активируется, но будет ожидать ручного запуска.

И выглядит это все в UI Gitlab как:

todo: screeshot from giltlab

## Conditions definition ##

Ключевым моментом всего подхода являются **CONDITIONS**.

FEATURE_KEYS, BUGFIX_KEYS, MR_KEYS, ... - это **CONDITIONS** т.е. предопределенные условия при которых pipeline запускается.

Для Gitflow pipeline они определены в файле [gitflow-pipeline-conditions.yml](./src/common/gitflow-pipeline-conditions.yml) и выглядят как:

```yaml
#
# conditions/rules for pipeline according to gitflow workflow.
#
variables:
  FEATURE_KEYS: ""
  BUGFIX_KEYS: ""
  MR_KEYS: ""
  DEVELOP_KEYS: ""
  RELEASE_KEYS: ""
  HOTFIX_KEYS: ""

.start-auto:
  variables:
    REGEX_KEY_AUTO: /\|(key-some|key-some-auto)\|/i
  rules:
    - if: $CI_COMMIT_REF_NAME =~ /^feature.*$/i && $CI_PIPELINE_SOURCE != "merge_request_event" && $FEATURE_KEYS =~ $REGEX_KEY_AUTO
    - if: $CI_COMMIT_REF_NAME =~ /^bugfix.*$/i && $CI_PIPELINE_SOURCE != "merge_request_event" && $BUGFIX_KEYS =~ $REGEX_KEY_AUTO
    - if: $CI_COMMIT_REF_NAME =~ /^develop.*$/i && $CI_PIPELINE_SOURCE != "merge_request_event" && $DEVELOP_KEYS =~ $REGEX_KEY_AUTO
    - if: $CI_COMMIT_REF_NAME =~ /^release.*$/i && $CI_PIPELINE_SOURCE != "merge_request_event" && $RELEASE_KEYS =~ $REGEX_KEY_AUTO
    - if: $CI_COMMIT_REF_NAME =~ /^hotfix.*$/i && $CI_PIPELINE_SOURCE != "merge_request_event" && $HOTFIX_KEYS =~ $REGEX_KEY_AUTO
    - if: $CI_MERGE_REQUEST_ID && $MR_KEYS =~ $REGEX_KEY_AUTO
    - when: never

.start-manual:
  variables:
    REGEX_KEY_MANUAL: /\|key-some-manual\|/i
  rules:
    - if: $CI_COMMIT_REF_NAME =~ /^feature.*$/i && $CI_PIPELINE_SOURCE != "merge_request_event" && $FEATURE_KEYS =~ $REGEX_KEY_MANUAL
      when: manual
    - if: $CI_COMMIT_REF_NAME =~ /^bugfix.*$/i && $CI_PIPELINE_SOURCE != "merge_request_event" && $BUGFIX_KEYS =~ $REGEX_KEY_MANUAL
      when: manual
    - if: $CI_COMMIT_REF_NAME =~ /^develop.*$/i && $CI_PIPELINE_SOURCE != "merge_request_event" && $DEVELOP_KEYS =~ $REGEX_KEY_MANUAL
      when: manual
    - if: $CI_COMMIT_REF_NAME =~ /^release.*$/i && $CI_PIPELINE_SOURCE != "merge_request_event" && $RELEASE_KEYS =~ $REGEX_KEY_MANUAL
      when: manual
    - if: $CI_COMMIT_REF_NAME =~ /^hotfix.*$/i && $CI_PIPELINE_SOURCE != "merge_request_event" && $HOTFIX_KEYS =~ $REGEX_KEY_MANUAL
      when: manual
    - if: $CI_MERGE_REQUEST_ID && $MR_KEYS =~ $REGEX_KEY_MANUAL
      when: manual
    - when: never
```

Ключевые моменты:

- Top level variables: FEATURE_KEYS, BUGFIX_KEYS, MR_KEYS, … - переменные которые декларируют все возможные CONDITIONs которые можно использовать в pipeline.
  - где имя переменной это произвольное имя CONDITION. в данном случае имена  CONDITIONs коррелируют с именами веток согласно Gitflow workflow.
  - при этом, переменные CONDITIONs  должны быть переопределены в файле соответствующего pipeline с указанием  |KEY-1|KEY-2|KEY-N|.
- .start-auto: and .start-manual: - объявлены как HIDDEN JOB и содержат только условия запуска pipeline, которые задаются с помощью инструкции [RULES](https://docs.gitlab.com/ee/ci/jobs/job_rules.html).
  - каждая из групп  .start-auto: or .start-manual: определяет несколько "- if:" для каждого CONDITION с указанием режима запуска [WHEN](https://docs.gitlab.com/ee/ci/yaml/#when).
  - далее эти условия запуска применяются к конкретному JOB с помощью инструкции  ["extends:"](https://docs.gitlab.com/ee/ci/yaml/#extends )
- Каждый IF определяет expression который можно трактовать как "если запушили коммит в ветку {BRANCH-NAME} и переменная ${BRANCH-NAME}_KEYS соответствует regex (содержит |KEY|), то вернуть true ", при этом regex задается через переменные уровня JOB $REGEX_KEY_MANUAL и $REGEX_KEY_AUTO .
- в свою очередь переменные $REGEX_KEY_MANUAL и $REGEX_KEY_AUTO должны быть переопределены в конкретном JOB и содержать regex который и определяет за каким |KEY| данный JOB закрепляется.  (подробнее см. [JOBs definition](#jobs-definition)).

> Необходимо понимать что файл gitflow-pipeline-conditions.yml является общим для всех других Gitflow pipeline и определяет только какие CONDITIONS есть и условия их запуска, а не сам pipeline. Сам pipeline декларируется в отдельном файле и должен переопределить все переменные СONDITIONS.  подробнее смотрите в [pipeline definition](#pipeline-definition).

## JOBs definition ##

JOBs  в общем виде задаются как (пример для |build|):

1. Определяем HIDDEN JOB  и закрепляем JOB за KEYs: |build| или |build-auto| или |build-manual|

```yaml
.job-build:
  stage: build
  script:
      - echo "execute script to build"
  variables:
    REGEX_KEY_AUTO: /\|(build|build-auto)\|/i
    REGEX_KEY_MANUAL: /\|build-manual\|/i
```

2. Далее определяем два VISIBLE JOBs для ручного и автоматического запуска как:

```yaml
build:manual:
  extends:
    - .start-manual
    - .job-build

build:auto:
  extends:
    - .start-auto
    - .job-build
```

В приведенном выше примере JOB задается в два шага.

<ins>На первом шаге закрепляем JOB за KEYs.</ins> Для этого в переменных  REGEX_KEY_AUTO и REGEX_KEY_MANUAL необходимо указать regex для какого |KEY| данный JOB будет реагировать при наступлении CONDITION.

Некоторые ключевые моменты:

- JOB определяем как [hidden(Start the job name with a dot (.))](https://docs.gitlab.com/ee/ci/jobs/#hide-jobs).
- имена |KEY| произвольные и задаются по своему усмотрению, но желательно что бы соответствовали определенному naming convention.
- имена |KEY| обязательно должны экранироваться символом "|"  с обеих сторон. такой подход позволяет использовать даже комментарии в описание pipeline.
  - "deploy-q|" or  "|deploy-q" or "deploy-q" - incorrect.
  - "|deploy-q|" - correct.
  - "|build|test|deploy-q|" - correct.
  - " some description: |deploy-q-manual|restart-q-manual|"  - correct.
- regex указываем в двух переменных. по сути мы переопределяем  значения этих переменных заданных в  gitflow-pipeline-conditions.yml.
  - REGEX_KEY_AUTO - для автоматического запуска JOB.
  - REGEX_KEY_MANUAL - для запуска JOB руками.
- за одним |KEY| можно закрепить несколько JOB. они будут стартовать параллельно.

<ins>На втором шаге определяем два VISIBLE JOBs.</ins>  Задаем им имена, т.к. они будут видны в UI (e.g. build:manual или build:auto). И с помощью инструкции  ["extends:"](https://docs.gitlab.com/ee/ci/yaml/#extends )  указываем какие hidden job переиспользовать. в нашем случае это режим запуска .start-manual или .start-auto и какой JOB это .job-build.

## Keys naming convention ##

Naming convention для текущих примеров:

- \*-auto или без -suffix - запуск JOB автоматически. по умолчанию*.
- \*-manual  - запуск JOB вручную.
- \*-skip - пропустить JOB (не выполнять).
- \*-d-\* - действия на DEV environment, например deploy/restart/clean/so on.
- \*-q-\* - действия на  QA (TEST) environment, например deploy.
- \*-p-\* - действия на  PROD environment, например deploy.

Вы можете согласовать свой naming convention и придерживаться его, например условиться что

- \*-manual или без -suffix  - запуск JOB вручную. по умолчанию*.
- \*-auto - запуск JOB автоматически.
- \*-test-\* или \*-t-\*  - действия на TEST environment.
- \*-s-\*  - действия на STAGE (pre-prod) environment.

 *по умолчанию - режим запуска по умолчанию, т.е.какой режим для |KEY| без suffix, например:

- если |build| == |build-auto| - режим запуска по умолчанию будет автоматический.
- если |build| == |build-manual| - режим запуска по умолчанию будет вручную.

## Pipeline definition ##

Файл [gitflow-pipeline-conditions.yml](src/common/gitflow-pipeline-conditions.yml) является общим для всех других Gitflow pipeline и определяет только какие CONDITIONS есть и условия их запуска, а не сам pipeline. Сам pipeline декларируется в отдельном файле и должен переопределить все переменные СONDITIONS (подробнее смотрите [Conditions definition](#conditions-definition)).  

Примеры pipelines на базе Gitflow pipeline conditions:

- [MR-Only](./src/mr-only/pipeline.yml) - pipeline только для Merge requests. Выполняет проверки: на корректное имя ветки и что автор комита не имеет права мержить.
- [Multi-Stage](./src/pipeline-sample/pipeline.yml) - пример организации pipeline для Dev, QA, Prod сред. Build docker images and deploy to docker-compose on remote host.

> Обратите внимание на организацию артефактов pipeline по файлам: pipeline.yml, jobs.yml, scripts.yml - свое рода это тоже conventions.

Для наглядности представления pipeline можно использовать возможности multiline string языка YAML (смотрите примеры по ссылке: <https://stackoverflow.com/a/21699210>).

Например pipeline можно представить как:

```yaml
  FEATURE_KEYS: >
    deploy-d: |deploy-d-manual|restart-d-manual| # some description ...
    deploy-q: |deploy-q-manual| also you can |restart-q-manual|
  BUGFIX_KEYS: $FEATURE_KEYS
  MR_KEYS: "|mr-rules-manual|"
  DEVELOP_KEYS: "|deploy-d-auto|deploy-q-auto|restart-d-manual|restart-q-manual|"
  RELEASE_KEYS: >
    deploy-d: |deploy-d-manual|restart-d-manual|
    deploy-q: |deploy-q-manual|restart-q-manual|
    deploy-p: |deploy-p-manual|restart-p-manual|
  HOTFIX_KEYS: $RELEASE_KEYS
  ```

  Главное придерживаться правила: *имена |KEY| обязательно должны экранироваться символом "|" с обеих сторон*

## Conclusion ##

В свое время передо мной стояла задача по реализации CI/CD pipeline. Изучив имеющиеся под рукой и в интернете реализации и почерпнув от туда удачные идеи я разработал описанный здесь подход.

Надеюсь что описанный здесь Gitflow pipeline окажется полезным и станет ценным источником вдохновения для других.
