# Gitflow workflow in short #

According to [Gitflow workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) a developer works with the next branches only:

- master or main
- develop (default)
- feature/*
- bugfix/*
- release/*
- hotfix/*

Gitflow Workflow:

- **develop** branch is created from **master**
- **release** branches are created from **develop**
- **feature** branches are created from **develop**
- When a **feature** is complete it is merged into the **develop** branch via **MR**, then deleted
- When a **release** branch is done, it is merged into **develop** and **master**, then deleted
  - If an issue in master is detected a **hotfix** branch is created from master
  - Once the **hotfix** is complete, it is merged to both **develop** and **master**, then delete

Related link: <https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow>
