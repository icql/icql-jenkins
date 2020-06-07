# icql-jenkins

jenkins pipeline（多分支流水线）

https://devops.jenkins.icql.work

详情介绍 [230_k8s-实践](https://icql.work/2020/04/29/200_k8s/230_k8s-%E5%AE%9E%E8%B7%B5/)

``` bash
├──icql
	├──readme-doc（文档）
	├──sync-gitee-github（gitee-github仓库同步）
	├──deploy-k8s（k8s通用资源部署）
	├──deploy-static（k8s静态资源部署）
	├──deploy-api（api服务CI/CD，TODO）
	├──cron-job（定时job，TODO）
```

revision-graph：
![revision-graph](img/revision-graph.png)