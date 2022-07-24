## Saipan Artifact Mirror

### Why This Project

在制品库领域，我们发现第三方软件制品和企业内部自主制品在性质上有很大不同，这个在 [Rethink Repository Management](https://aitchjoe.wordpress.com/2019/11/15/rethink-repository-management/) 有详细讨论，简单概述：

* 第三方制品通常不会直接保存在企业内部的制品库，而是以制品镜像的形式存在。
* 第三方制品通常是公开的、可自由获取；而内部制品大多不对外公开、即使对不同的内部用户也往往有细粒度的权限控制。
* 每个技术领域的主流第三方制品库通常数量有限、启用镜像基本是一次性配置；而企业的内部制品库虽然可能也是一个，但其下的 Tenant / Organization / Project / Group 数量会持续增长、用户使用会不断变更等等等等，因此需要足够的二级管理功能。
* 第三方制品通常有新旧版本同时被多个用户使用；而对于非软件提供商的企业，除了少量共享库，应用制品往往只使用最新版本、旧版本的保存价值极小。
* 在 UI 上，前者的视角直接就是制品，而后者首先是 Project 其下才是制品。

这些性质上的差别决定了在产品设计上会产生相互矛盾的倾向，而在用户的运维上也会有重大不同；如果仅仅是运维第三方制品的镜像，除了配置信息、运维方实际是不用担心丢失任何缓存数据的。而现在制品库的主流产品如 JFrog、Nexus 将两者混为一谈，甚至从 License 上限制了分开部署，实际上是干扰了运维。

本项目的定位仅仅是作为第三方制品的企业内部镜像，通过以下设计极大简化在这方面的运维，虽然没有解决自主制品管理的复杂性，但是将特定问题拆分解决应该是一个正道。

### Design

设计上作如下考虑：

* Keep Less：
  * 以上的拆分和聚焦，就是最主要的 Keep Less。
  * 绝对不会像 JFrog 一样扩展到自己不擅长的 DevOps 领域。
  * 产品本身不作登录或权限控制，采用类似 Thanos 或 Loki 的 Sidecar 模式由其他软件代理甚至如下处理：
    * Admin UI 不需登录，所有的变更操作只是生成配置文件、由管理员在 Git 上 Commit（自然也就由 Git 做了权限控制）。
    * 但还有非配置变更类的管理操作比如 Invalid cache 等，如不考虑稽核可由管理员在后台操作实现，否则只能启用 Sidecar 模式。
  * HTTPS 依赖于前置 LB。
* Out of the Box：对于常见第三方主流制品库默认提供默认配置，勿需手工进行。
  * 注意部分制品库虽为公开，但仍要求提供访问 Token 即使可以免费创建，比如 [registry.redhat.io](https://registry.redhat.io/)，还有 Docker Hub 的限流情况等等；因此这些需要由用户补充配置；也因为这个原因，不能完全使用以下的 COC 方式实现。
* Opinionated：
  * 内部访问的 URL 勿需用户设计，直接使用如 `https://saipan.example.com/docker-io` 或者 `https://saipan.example.com/maven/central`。
  * JFrog 的 Virtual Repository 或 Nexus 的 Repository Group 很容易产生误用，因此不提供相应功能。
* COC：
  * 例如类似 JFrog 的 Generic Repository，但是勿需配置，对于 `https://saipan.example.com/web/download-java-net/**` 即可自动获取相应的 `download.java.net/**` 内容。
    * 如果担忧滥用可以通过黑名单、白名单等控制，由本产品或社区提供主流软件下载网站清单。
    * 以上开箱即用的部分也可以采用 COC 而非默认配置的方式实现，但总归由产品方提供而非用户操心。
* GitOps：所有配置变更可以通过 Git 管理，产品端只需配置相应 Git URL 即可自动同步。
* HA：由于产品定位及以上的设计，部署多个实例并通过 LB 暴露即可实现 HA，实例间勿需同步任何内容。
* Warm up：新增实例或者灾难恢复时，可以通过产品持续统计结果所生成的 Warm up 脚本实现预加载。
* Compromise：有部分第三方公共制品因为各种原因并不能通过镜像供企业内部直接使用，仍需手工转存到内部制品库后再使用；虽然从属性上来说它属于第三方制品、应该由本产品管理，但这样产品就需要考虑数据持久化及相应的更多运维因素、而无法简单的 GitOps，因此对于这部分内容产品在设计上将舍弃、仍交由内部制品库软件管理。

### Roadmap

目前处于内部开发阶段：[aitchjoe/saipan-artifact-mirror](https://github.com/aitchjoe/saipan-artifact-mirror)。

### About

* Project Code：Saipan 是 2019 年疫情前的最后一次旅游地。
* Mirror：按业界主流的提法，从技术实现上来讲本产品似乎应该叫 Proxy 而非 Mirror（参见 JFrog 关于 [Proxy vs. Mirror](https://www.jfrog.com/confluence/display/JFROG/Repository+Management#RepositoryManagement-RemoteRepositories) 的说明），但是从用户视角，制品的 Proxy 和日常上网的 Proxy 其实有很大差别（后者在配置后对于用户基本就是透明的）；另外虽然对于整个外部制品网站本产品应该算 On demand 的 Proxy，但具体到每个制品、用户所使用的就是外部制品的内部 Mirror。而不管这些咬文嚼字，实际上最终决定产品的名称是因为没法简称为 SAP。
