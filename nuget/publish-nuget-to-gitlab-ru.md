# Как настроить публикацию nuget пакетов в реест Gitlab #

> How to configure publishing nuget packages into Gitlab nuget registry

В этой статье приведены краткие инструкции по настройке публикации nuget пакетов в Gitlab для проектов Net Core. Во второй части статьи представлен bonus-скрипт для автоматизации процесса публикации.

Современные Net Core проекты редко обходятся без nuget пакетов, и большинство разработчиков на C# хотя бы раз сталкивались с необходимостью или желанием опубликовать свои собственные nuget пакеты, а затем использовать их в других местах разрабатываемой системы.

Для решения этой задачи необходимо ответить на два вопроса:

- Первый вопрос: где публиковать nuget пакеты? особенно на этапе разработки и отладки. Публикация в общедоступный реестр nuget.org идея не очень. Также использование внешних/облачных ресурсов может быть запрещена политикой безопасности.
- Второй вопрос: как публиковать? что для этого надо?

Оба вопроса являются блокирующими, и могут вызывать трудности у разработчиков при их решении.

В первом вопросе: поиск ресурсов возможно потребует вовлечения более большего круга лиц, менеджеров, devops специалистов, админов всея руси..., и процесс может затянуться.

Во втором вопросе: для процесса публикации д.б. экспертиза, собственно, как это делать, а если ее нет, то нужно время на изучение документации, статьей, постов, выполнение локальной отладки и т.п.

![angry developer](angry-developer.jpeg)

Но если вы используете GitLab в качестве основного репозитория исходного кода, то вам повезло! Возможно не все знают, но в GitLab, даже в Community Edition,  также имеется [реестр для nuget/npm/etc пакетов](https://docs.gitlab.com/ee/user/packages/nuget_repository/), который очень удобно использовать для целей разработки и отладки nuget/npm/etc пакетов. [Аналогичный реестр есть и у Github](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-nuget-registry).

## Краткие инструкции ##

Чтобы иметь возможность публиковать и восстанавливать свои пакеты nuget в/из Gitlab registry, нам необходимо выполнить следующие шаги:

1. Создание deploy token
2. Сборка и публикация nuget пакета в Gitlab registry (pack and publish)
3. Создание и настройка проектного nuget.config

### Создание deploy token ###

Шаги по созданию через UI: <https://docs.gitlab.com/ee/user/project/deploy_tokens/index.html#create-a-deploy-token>

При настройке token надо выбрать следующие scopes: "read_package_registry" и "write_package_registry".

Рекомендуется использовать deploy token уровня группы, в этом случае deploy token можно применять для всех проектов/репозиториев группы. Для создания такого токена требуется роль Владелец группы (owner).
Также можно использовать deploy token для уровня проекта, но такой токен дает доступ только для одного конкретного проекта.

Для примера мы создали следующий deploy token:

```
Name: Gitlab-nuget-deploy-token
Username: Gitlab-nuget-deploy-user
Expiration date: <empty>
Token: gldt-u2V99BeUeWMyoouUS3uL
```

### Сборка и публикация nuget пакета в Gitlab registry (pack and publish) ###

Выполните в bash:

```sh
cd {root of *.csprj folder}

dotnet pack

cd ./bin/Release

dotnet nuget push *.nupkg --source https://<company.com>/api/v4/projects/<your_project_id>/packages/nuget/index.json  --api-key gldt-u2V99BeUeWMyoouUS3uL
```

где <your_project_id> - project id репозитория проекта. Как найти project id см: <https://docs.gitlab.com/ee/user/project/working_with_projects.html#access-a-project-by-using-the-project-id>

### Создание и настройка проектного nuget.config ###

В проектах, где нам необходимо восстановить наши nuget пакеты, выполняем:

```sh
cd {root folder for repo or group of repos}

dotnet new nugetconfig

dotnet nuget add source "https://<company.com>/api/v4/groups/<group_id>/-/packages/nuget/index.json" --name GitlabNugetRepo --username Gitlab-nuget-deploy-user --password gldt-u2V99BeUeWMyoouUS3uL --configfile "nuget.config"
```

где <group_id> - id группы. Как найти id группы см: <https://docs.gitlab.com/ee/user/group/#get-the-group-id>

## Автоматизация процесса публикации ##

Для автоматизации процесса публикации мы можем написать help-script.

Предположим, у нас есть следующая структура папок проектов, и мы хотим опубликовать NuGet пакет для каждого проекта.

```
   |-projects
   |---Audit
   |---Audit.Abstractions
   |---Audit.Transport.Db
   |---Audit.Transport.Http
   |---publish_nuget.sh
```

Тогда мы можем использовать следующий скрипт:

<details>

<summary>Please expand to see the script code</summary>

```sh
#!/bin/bash

API_URL="https://<company.gitlab.com>/api/v4/projects/<project_id>/packages"
NUGET_REGISTRY=$API_URL/nuget/index.json
API_KEY=$AuditLib_Maintainer_Token
NUGET_TOKEN=$AuditLib_Nuget_Token

function remove_previous_pkg
{
     if [[ -z "$API_KEY" ]]; then
        echo "API_KEY is absent. Deleting nugets are skipped."
        return
    fi

    FULL_FILE_NAME=$(basename -- *.nupkg)
    PACKAGE_NAME=$(echo "$FULL_FILE_NAME" | sed -E 's/(.*)\.[0-9]+\.[0-9]+\.[0-9]+.*.nupkg/\1/')
    PACKAGE_VERSION=$(echo "$FULL_FILE_NAME" | sed -E 's/.*\.([0-9]+\.[0-9]+\.[0-9]+.*).nupkg/\1/')

    echo "PACKAGE_NAME: $PACKAGE_NAME"
    echo "PACKAGE_VERSION: $PACKAGE_VERSION"

    RESPONSE=$(curl -s -H "Authorization: Bearer $API_KEY" "$API_URL?package_name=$PACKAGE_NAME&package_version=$PACKAGE_VERSION")

    PACKAGE_ID=$(echo "$RESPONSE" | grep -Eo '"id":([0-9]+),"name":"'"$PACKAGE_NAME"'"' | grep -Eo '[0-9]+')

    echo "PACKAGE_ID: $PACKAGE_ID"

    if [[ -z "$PACKAGE_ID" ]]; then
           echo "Package is absent in Nuget repository"
       else
           curl --request DELETE --header "PRIVATE-TOKEN: $API_KEY" "$API_URL/$PACKAGE_ID"
           echo "Package in Nuget repository was deleted"
    fi
}

function pack_and_publish
(
    echo "----------- Publishing  $1 statred -----------"

    rm $1/bin/Release/*.nupkg

    cd $1

    dotnet pack

    cd ./bin/Release

    remove_previous_pkg

    dotnet nuget push *.nupkg --source $NUGET_REGISTRY --api-key $NUGET_TOKEN

    printf "\n\n"
)

pack_and_publish ./audit
pack_and_publish ./Audit.Abstractions
pack_and_publish ./Audit.Transport.Http
pack_and_publish ./Audit.Transport.Db
```

</details>

Этот скрипт необходимо запускать под bash ([git bash terminal под windows](https://www.atlassian.com/git/tutorials/git-bash)).

Этот скрипт также удаляет ранее опубликованный пакет nuget. Если имя и версия пакетов одинаковые, то такой nuget пакет сначала будет удален из Gitlab. Это удобно, когда мы выполняем отладку текущего кода, входящего в пакет, и не хотим заморачиваться с поднятиями версий.

> Также после перепубликации пакетов **не забудьте** сбросить local nuget cache как ```dotnet nuget locals all --clear``` (или другим способом см: <https://support.syncfusion.com/kb/article/6265/how-to-clear-nuget-cache>) или, в случае с docker compose, выполнить rebuild Docker images with out cache как ```docker-compose down && docker-compose build --no-cache``` или перед запуском docker compose просто удалить предыдущие docker images.

Ну и конечно же, этот скрипт можно адаптировать для целей CI/CD взяв его за основу.

### Настройка environment vars ###

В целях безопасности этот скрипт использует два вида токенов Gitlab и получает их из переменных окружения (environment vars), чтобы скрыть настоящие токены из исходного кода хранящегося в репозитории. В связи с чем, перед запуском скрипта вам следует установить две переменные окружения:

```
AuditLib_Nuget_Token: {nuget-token}
AuditLib_Maintainer_Token: {maintainer_token}
```

где:

- `{nuget-token}`- это deploy access token с правами `read_package_registry` и  `write_package_registry`. [Инструкция по созданию](https://docs.gitlab.com/ee/user/project/deploy_tokens/index.html#create-a-deploy-token).
- `{maintainer_token}` - это personal access token пользователя с правами maintainer (используется для удаления nuget packages).

#### Для Windows ####

В **cmd** выполните следующие команды:

```cmd
setx AuditLib_Nuget_Token {nuget-token}
setx AuditLib_Maintainer_Token {maintainer_token}
```

#### Для Mac/Unix ####

Необходимо отредактировать `~/.zshrc` (Mac) или `~/.bashrc` (bash for unix) и добавить следующие строки:

```sh
export AuditLib_Nuget_Token={nuget-token}
export AuditLib_Maintainer_Token={maintainer_token}
```

#### Проверка vars ####

Перезапустить VSCode или terminal и проверить:

```sh
echo $AuditLib_Nuget_Token
echo $AuditLib_Maintainer_Token
```

переменные должны содержать tokens.

## PS ##

В свое время и я стакнулся с этой задачей, на тот момент у меня не было необходимой экспертизы, я потратил несколько часов на ее получение, а также на написание и отладку скрипта, и что бы не потерять полученную экспертизу я решил закрепить ее в этой статье.
Я искренне надеюсь, что эта статья поможет кому-то сэкономить время и силы при решении аналогичных задач.

Полезные ссылки:

- <https://docs.gitlab.com/ee/user/packages/nuget_repository/>
- <https://learn.microsoft.com/en-us/nuget/quickstart/create-and-publish-a-package-using-the-dotnet-cli>
- <https://docs.gitlab.com/ee/user/project/deploy_tokens/index.html#create-a-deploy-token>
- <https://docs.gitlab.com/ee/user/group/#get-the-group-id>
- <https://docs.gitlab.com/ee/user/project/working_with_projects.html#access-a-project-by-using-the-project-id>
