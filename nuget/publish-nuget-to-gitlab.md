# How to configure publishing nuget packages to Gitlab #

This article describes a way and short instructions how to configure publishing nuget packages to Gitlab for Net Core projects. At the second part of this article there is a bonus script to automate the process of publishing.

The modern Net Core projects sure rarely do without nuget packages and most of c# developers at least once faced with needs or desire to publish their own nuget packages and then use them in other parts of the system.

- The first  question is "where is the place to publish nuget packages?", especially at the developing and debugging stage, publishing to the public nuget.org registry is not a good idea.
- and the second question is "how to publish?".

Both questions are blockers and the developers prefer to postpone this task and usually trying go by other ways: using mono repository or use git submodules or even copy past a ready dll to source code because the publishing nuget packages cause difficulties in achieving.

![angry developer](angry-developer.jpeg)

But if you use GitLab as the main source-code repository there is a solution! Because Gitlab also has a nuget/npm/etc registry and you can use it!

## Short instructions ##

To have ability to publish and restore our nuget packages to/from Gitlab we need to do the next steps:

    1. Create a deploy token
    2. Pack and publish nuget package to Gitlab registry
    3. Create and configure nuget.config

### Create a deploy token ###

Steps to create via UI: <https://docs.gitlab.com/ee/user/project/deploy_tokens/index.html#create-a-deploy-token>

Please set scopes: "read_package_registry" and "write_package_registry".

> Recommended to create a GROUP deploy token, for that you must have the Owner role for the group. That token is acceptable for all projects/repos of the group. For Project level is also possible but that token is acceptable for single project only.

For example we have created the next deploy token:

```
Name: Gitlab-nuget-deploy-token
Username: Gitlab-nuget-deploy-user
Expiration date: <empty>
Token: gldt-u2V99BeUeWMyoouUS3uL
```

### Pack and publish a nuget package to Gitlab registry ###

```sh
cd {root of *.csprj folder}

dotnet pack

cd ./bin/Release

dotnet nuget push *.nupkg --source https://hq-git.auriga.ru/api/v4/projects/<your_project_id>/packages/nuget/index.json  --api-key gldt-u2V99BeUeWMyoouUS3uL
```

where <your_project_id> - project id of project repo. How to find a project id see: <https://docs.gitlab.com/ee/user/project/working_with_projects.html#access-a-project-by-using-the-project-id>

### Create and configure nuget.config ###

For projects where we need to restore our nuget packages:

```sh
cd {root folder for repo or group of repos}

dotnet new nugetconfig

dotnet nuget add source "https://hq-git.auriga.ru/api/v4/groups/<group_id>/-/packages/nuget/index.json" --name GitlabNugetRepo --username Gitlab-nuget-deploy-user --password gldt-u2V99BeUeWMyoouUS3uL --configfile "nuget.config"
```

where <group_id> - id of group of project repos. How to find a group id of projects see: <https://docs.gitlab.com/ee/user/group/#get-the-group-id>

## Automate process of publishing ##

For automating the process of publishing nuget packages we can use a help-script.

Assume we have the next folder structure of projects and want to publish a nuget package for each project.

```
   |-projects
   |---Audit
   |---Audit.Abstractions
   |---Audit.Transport.Db
   |---Audit.Transport.Http
   |---publish_nuget.sh
```

Then we can run the next script:

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

This script should be run under bash (git bash terminal under windows).

This script also does remove a previously published nuget package. This action is useful when you do a debugging the current code of a package and want to update this one by an upgraded version.

Also after republishing **don't forget** reset a local nuget cache (```dotnet nuget locals all --clear```, or see <https://support.syncfusion.com/kb/article/6265/how-to-clear-nuget-cache>) or rebuild Docker images with out cache (ie: ```docker-compose down && docker-compose build --no-cache```).

And of course we might adopt this script for CI/CD purposes.

### Configure environment vars ###

For security reasons this script uses two kinds of Gitlab tokens and gets them from environment vars to hide the real tokens from source code. Thus before run script you should set two Environment vars:

```
AuditLib_Nuget_Token: {nuget-token}
AuditLib_Maintainer_Token: {maintainer_token}
```

where:

- {nuget-token}- it is a 'deploy access token' with 'read_package_registry' and 'write_package_registry' permissions.
- {maintainer_token} - it is a 'personal access token' by user with a maintainer role for the Gitlab project repository. (this is required for nuget packages deleting).

#### For Windows ####

Under cmd run the next commands:

```cmd
setx AuditLib_Nuget_Token {nuget-token}
setx AuditLib_Maintainer_Token {maintainer_token}
```

#### For Mac/Unix ####

We need to  edit ~/.zshrc (Mac) or ~/.bashrc (bash for unix) and append the next lines:

```sh
export AuditLib_Nuget_Token={nuget-token}
export AuditLib_Maintainer_Token={maintainer_token}
```

#### Test vars ####

Restart VSCode or terminal and check:

```sh
echo $AuditLib_Nuget_Token
echo $AuditLib_Maintainer_Token
```

The vars should have the tokens.

## Conclusion ##

I spent several hours reading various articles, as well as writing and debugging the script, and I hope this article will help someone save time and effort when solving similar tasks.

Useful links:

- <https://docs.gitlab.com/ee/user/packages/nuget_repository/>
- <https://learn.microsoft.com/en-us/nuget/quickstart/create-and-publish-a-package-using-the-dotnet-cli>
- <https://docs.gitlab.com/ee/user/project/deploy_tokens/index.html#create-a-deploy-token>
- <https://docs.gitlab.com/ee/user/group/#get-the-group-id>
- <https://docs.gitlab.com/ee/user/project/working_with_projects.html#access-a-project-by-using-the-project-id>
