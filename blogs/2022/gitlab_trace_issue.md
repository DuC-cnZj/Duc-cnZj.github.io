---
title: gitlab the job has no trace 问题
date: '2022-01-25 15:20:38'
sidebar: false
categories:
 - 技术
tags:
 - gitlab
publish: true
---

# gitlab the job has no trace 问题

真的是蛋疼的问题，影响了好几个月，ci/cd日志丢失

说下解决方法

1. 引起原因是因为数据的 ci_job_artifacts created_at 时间不正确，gitlab拿不到

2. 如果你本地是时区修改了的话，db的时区必须要是 UTC

3. [Artifacts stored/searched in a wrong date directory - System Administration - GitLab Forum](https://forum.gitlab.com/t/artifacts-stored-searched-in-a-wrong-date-directory/51753)

4. https://gitlab.com/gitlab-org/gitlab/-/issues/26320

## 最终解决方案

把 ci_job_artifacts 表的 created_at 类型从 timestamptz 改成 timestamp

我没有试过把pg的时区改成UTC, 也许也可以
