---
title: sonarqube-community-branch-plugin 对 GitLab MR 评论的异常
tags: SonarQube sonarqube-community-branch-plugin
key: sonarqube-community-branch-plugin-post-commit-status-not-found
---
[sonarqube-community-branch-plugin](https://github.com/mc1arke/sonarqube-community-branch-plugin) 是一个可以让 SonarQube Community 版支持分支功能的插件，但是在通过此插件对 GitLab Merge Request 进行分析时有时无法将检测出的问题以评论的方式输出到 MR 上。

<!--more-->

## 问题
因为这个插件将检测出的问题输出到 MR 评论的时机是在 SonarQube CE 分析完成后，所以我们可以在 SonarQube 的 ce.log 看到这个插件输出的日志。
在分析完成后，发现实际上在 [Post commit build status](https://docs.gitlab.com/ee/api/commits.html#post-the-build-status-to-a-commit) 时就被一个 404 Not Found 中止了接下来的评论逻辑。  
按照经验来说，MR 的 Commit 就算未合入，通过 MR 目标仓库的 API 也可以找到 Commit 的信息。
而且这个问题在源分支和目标分支同项目时就不会触发，只有通过 Fork 出来的仓库向目标仓库提交 MR 时才会触发这个问题。

## 解决
经过几次尝试，发现这个 API 实际上是用来设置 Commit 对应的 Pipeline 状态的。
而对于 [MR Pipeline](https://docs.gitlab.com/ee/ci/merge_request_pipelines) 来说，GitLab 有以下解释
>Pipelines for merge requests can run when you:
* Create a new merge request.
* Commit changes to the source branch for the merge request.
* Select the Run pipeline button from the Pipelines tab in the merge request. 

那么也就可以解释清楚为什么会 404 Not Found 了，因为对 GitLab MR 来说，MR 绑定的 Pipeline 实际上是在源仓库运行的 Pipeline，通过目标仓库的 API 自然是设置不了源仓库的 Pipeline 状态的。

这个问题实际上只需要修改以下代码即可解决
```java
final String statusUrl = projectURL + String.format("/statuses/%s", revision);
```
更改为
```java
final String statusUrl = sourceProjectURL + String.format("/statuses/%s", revision);
```
这个问题已在 [此 PR](https://github.com/mc1arke/sonarqube-community-branch-plugin/pull/308) 中得到解决，从 1.7.0 版本开始此问题已被修复。
