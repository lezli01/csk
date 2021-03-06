# 使用GitLab CI在Kubernetes服务上运行GitLab Runner并执行Pipeline {#task_1681380 .task}

本文主要演示如何在Kubernetes集群中安装、注册GitLab Runner，添加Kubernetes类型的executor来执行构建，并以此为基础完成一个Java源码示例项目从编译构建、镜像打包到应用部署的CICD过程。

本文以构建一个Java软件项目并将其部署到阿里云容器服务Kubernetes集群中为例，说明如何使用GitLab CI在阿里云Kubernetes服务上运行GitLab Runner、配置Kubernetes类型的executor，并执行Pipeline。

## 创建GitLab源码项目并上传示例代码 {#section_06v_dhf_8av .section}

1.  创建GitLab源码项目。 本示例中创建的GitLab源码项目地址为：

    ``` {#codeblock_6k5_2ch_e2w}
    http://xx.xx.xx.xx/demo/gitlab-java-demo.git
    ```

2.  执行以下命令获取示例代码并上传至GitLab。 

    ``` {#codeblock_7ek_sw1_50y}
    $ git clone https://code.aliyun.com/CodePipeline/gitlabci-java-demo.git
    $ git remote add gitlab http://xx.xx.xx.xx/demo/gitlab-java-demo.git
    $ git push gitlab master
    ```


## 在Kubernetes集群中安装GitLab Runner {#section_qa9_xyu_stk .section}

1.  获取GitLab Runner的注册信息。 
    1.  获取项目专用Runner的注册信息。 
        1.  登录GitLab。
        2.  在顶部导航栏中，选择**Projects** \> **Your projects**。
        3.  在**Your projects**页签下，选择相应的project。
        4.  在左侧导航栏中，选择**Settings** \> **CI / CD**。
        5.  单击**Runners**右侧的**Expand**。

            ![Expand](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123018/156574904738533_zh-CN.png)

        6.  获取URL和registration token信息。

            ![token信息](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123018/156574904738535_zh-CN.png)

    2.  获取Group Runners的注册信息。 
        1.  在顶部导航栏中，选择**Groups** \> **Your groups**。
        2.  在**Your groups**页签下，选择相应的group。
        3.  在左侧导航栏中，选择**Settings** \> **CI / CD**。
        4.  单击**Runners**右侧的**Expand**。

            ![获取注册信息](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123018/156574904738536_zh-CN.png)

        5.  获取URL和registration token信息。

            ![获取注册的token信息](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123018/156574904738537_zh-CN.png)

    3.  获取Shared Runners的注册信息。 

        **说明：** 只有管理员有权限执行此步操作。

        1.  在顶部导航栏中，单击![设置](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123018/156574904838538_zh-CN.png)进入Admin Area页面。
        2.  在左侧导航栏中，选择**Overview** \> **Runners**。
        3.  获取URL和registration token信息。

            ![runners的注册信息](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123018/156574904838539_zh-CN.png)

2.  执行以下命令获取并修改GitLab Runner的Helm Chart。 

    ``` {#codeblock_j6o_hea_wt7}
    git clone https://github.com/haoshuwei/ack-gitlab-runner.git
    ```

    修改values.yaml文件, 示例如下：

    ``` {#codeblock_it2_x06_kwu}
    ## GitLab Runner Image
    ##
    image: gitlab/gitlab-runner:alpine-v12.1.0
    
    ## Specify a imagePullPolicy
    ##
    imagePullPolicy: IfNotPresent
    
    ## Default container image to use for initcontainer
    init:
      image: busybox
      tag: latest
    
    ## The GitLab Server URL (with protocol) that want to register the runner against
    ##
    gitlabUrl: http://xx.xx.xx.xx/
    
    ## The Registration Token for adding new Runners to the GitLab Server. This must
    ## be retreived from your GitLab Instance.
    ##
    runnerRegistrationToken: "AMvEWrBTBu-d8czEYyfY"
    ## Unregister all runners before termination
    ##
    unregisterRunners: true
    
    ## Configure the maximum number of concurrent jobs
    ##
    concurrent: 10
    
    ## Defines in seconds how often to check GitLab for a new builds
    ##
    checkInterval: 30
    
    ## For RBAC support:
    ##
    rbac:
      create: true
      clusterWideAccess: false
    
    ## Configure integrated Prometheus metrics exporter
    ##
    metrics:
      enabled: true
    
    ## Configuration for the Pods that that the runner launches for each new job
    ##
    runners:
      ## Default container image to use for builds when none is specified
      ##
      image: ubuntu:16.04
    
      ## Specify the tags associated with the runner. Comma-separated list of tags.
      ##
      tags: "k8s-runner"
    
      ## Run all containers with the privileged flag enabled
      ## This will allow the docker:dind image to run if you need to run Docker
      ## commands. Please read the docs before turning this on:
      ##
      privileged: true
    
      ## Namespace to run Kubernetes jobs in (defaults to the same namespace of this release)
      ##
      namespace: gitlab
    
      cachePath: "/opt/cache"
    
      cache: {}
      builds: {}
      services: {}
      helpers: {}
    
    resources: {}
    ```

3.  执行以下命令安装GitLab Runner。 

    ``` {#codeblock_emc_f84_r9y}
    $ helm package .
    Successfully packaged chart and saved it to: /root/gitlab/gitlab-runner/gitlab-runner-0.1.37.tgz
    
    $ helm install --namespace gitlab --name gitlab-runner *.tgz
    ```

    查看相关的deployment/pod是否成功启动。若成功启动，则可在GitLab上看到注册成功的GitLab Runner，如下图所示：

    ![GitLab Runner](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123018/156574904838540_zh-CN.png)


## 缓存配置 {#section_qjb_agl_79c .section}

GitLab Runner对缓存方案的支持有限，所以您需要使用挂载volume的方式做缓存。在上面的示例中，安装GitLab Runner时默认使用/opt/cache目录作为缓存空间。您也可以通过修改values.yaml文件中的`runners.cachePath`字段修改缓存目录。

例如，如需建立maven缓存，您可以在`variables`下添加`MAVEN_OPTS`变量并指定本地缓存目录：

``` {#codeblock_664_rti_v4m}
variables:
  KUBECONFIG: /etc/deploy/config
  MAVEN_OPTS: "-Dmaven.repo.local=/opt/cache/.m2/repository"
```

如需挂载新的volume，您可以修改templates/configmap.yaml文件中的如下字段：

``` {#codeblock_wfr_4ge_91w}
cat >>/home/gitlab-runner/.gitlab-runner/config.toml <<EOF
            [[runners.kubernetes.volumes.pvc]]
              name = "gitlab-runner-cache"
              mount_path = "{{ .Values.runners.cachePath }}"
EOF
```

即在GitLab Runner进行register后，run之前，修改config.toml的配置。

## 设置全局变量 {#section_f4a_a2m_dhb .section}

1.  在GitLab的顶部导航栏中，选择**Projects** \> **Your projects**。
2.  在**Your projects**页签下，选择相应的project。
3.  在左侧导航栏中，选择**Settings** \> **CI / CD**。
4.  单击**Variables**右侧的**Expand**。添加GitLab Runner可用的环境变量。本示例中，添加以下三个变量： 

    ![环境变量](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123018/156574904838541_zh-CN.png)

    -   REGISTRY\_USERNAME：镜像仓库用户名。
    -   REGISTRY\_PASSWORD：镜像仓库密码。
    -   kube\_config：KubeConfig的编码字符串。
    执行以下命令生成KubeConfig的编码字符串：

    ``` {#codeblock_nmc_tjg_sp6}
    echo $(cat ~/.kube/config | base64) | tr -d " "
    ```


## 编写.gitlab-ci.yml {#section_sxq_9km_5cs .section}

编写.gitlab-ci.yml文件，完成java demo源码项目的编译构建、镜像推送和应用部署（可参考gitlabci-java-demo源码项目中的.gitlab-ci.yml.example）。

.gitlab-ci.yml示例如下：

``` {#codeblock_r1n_7js_2nc}
image: docker:stable
stages:
  - package
  - docker_build
  - deploy_k8s
variables:
  KUBECONFIG: /etc/deploy/config
mvn_build_job:
  image: registry.cn-beijing.aliyuncs.com/codepipeline/public-blueocean-codepipeline-slave-java:0.1-63b99a20
  stage: package
  tags:
    - k8s-runner
  script:
    - mvn package -B -DskipTests
    - cp target/demo.war /opt/cache
docker_build_job:
  image: registry.cn-beijing.aliyuncs.com/codepipeline/public-blueocean-codepipeline-slave-java:0.1-63b99a20
  stage: docker_build
  tags:
    - k8s-runner
  script:
    - docker login -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD registry.cn-beijing.aliyuncs.com
    - mkdir target
    - cp /opt/cache/demo.war target/demo.war
    - docker build -t registry.cn-beijing.aliyuncs.com/gitlab-demo/java-demo:$CI_PIPELINE_ID .
    - docker push registry.cn-beijing.aliyuncs.com/gitlab-demo/java-demo:$CI_PIPELINE_ID
deploy_k8s_job:
  image: registry.cn-beijing.aliyuncs.com/codepipeline/public-blueocean-codepipeline-slave-java:0.1-63b99a20
  stage: deploy_k8s
  tags:
    - k8s-runner
  script:
    - mkdir -p /etc/deploy
    - echo $kube_config |base64 -d > $KUBECONFIG
    - sed -i "s/IMAGE_TAG/$CI_PIPELINE_ID/g" deployment.yaml
    - cat deployment.yaml
    - kubectl apply -f deployment.yaml
```

.gitlab-ci.yml定义了一个Pipeline， 分三个阶段步骤执行：

``` {#codeblock_8f3_3nn_s0u}
image: docker:stable  # Pipeline中各个步骤阶段的构建镜像没有指定时， 默认使用docker:stable镜像
stages:
  - package                # 源码打包阶段
  - docker_build         # 镜像构建和打包推送阶段
  - deploy_k8s           # 应用部署阶段
variables:
  KUBECONFIG: /etc/deploy/config   # 定义全局变量KUBECONFIG
```

-   maven源码打包阶段：

    ``` {#codeblock_3i7_5j9_n3i}
    mvn_build_job:     # job名称
      image: registry.cn-beijing.aliyuncs.com/codepipeline/public-blueocean-codepipeline-slave-java:0.1-63b99a20  # 本阶段构建使用的构建镜像
      stage: package      # 关联的阶段名称
      tags:                     # GitLab Runner的tag
        - k8s-runner
      script:
        - mvn package -B -DskipTests  # 执行构建脚本
        - cp target/demo.war /opt/cache   # 构建物保存至缓存区
    ```

-   镜像构建和打包推送阶段：

    ``` {#codeblock_hey_e0f_oen}
    docker_build_job:  # job名称
      image: registry.cn-beijing.aliyuncs.com/codepipeline/public-blueocean-codepipeline-slave-java:0.1-63b99a20 # 本阶段构建使用的构建镜像
      stage: docker_build  # 关联的阶段名称
      tags:                      # GitLab Runner的tag
        - k8s-runner
      script:
        - docker login -u REGISTRY_USERNAME -p $REGISTRY_PASSWORD registry.cn-beijing.aliyuncs.com   # 登录镜像仓库
        - mkdir target
        - cp /opt/cache/demo.war target/demo.war
        - docker build -t registry.cn-beijing.aliyuncs.com/gitlab-demo/java-demo:$CI_PIPELINE_ID .     # 打包Docker镜像，使用的tag为本次Pipeline的ID
        - docker push registry.cn-beijing.aliyuncs.com/gitlab-demo/java-demo:$CI_PIPELINE_ID      # 推送Docker镜像
    ```

-   应用部署阶段：

    ``` {#codeblock_bdj_q3n_ajq}
    deploy_k8s_job:   # job名称
      image: registry.cn-beijing.aliyuncs.com/codepipeline/public-blueocean-codepipeline-slave-java:0.1-63b99a20   # 本阶段构建使用的构建镜像
      stage: deploy_k8s   # 关联的阶段名称
      tags:                      # GitLab Runner的tag
        - k8s-runner
      script:
        - mkdir -p /etc/deploy
        - echo $kube_config |base64 -d > $KUBECONFIG   # 配置连接Kubernetes集群的config文件
        - sed -i "s/IMAGE_TAG/$CI_PIPELINE_ID/g" deployment.yaml  # 动态替换部署文件中的镜像tag
        - kubectl apply -f deployment.yaml
    ```


## 执行Pipeline {#section_1y0_7hj_pjv .section}

提交.gitlab-ci.yml文件后，Project gitlab-java-demo会自动检测到这个文件并自行Pipeline， 如下图所示：

![执行Pipeline](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123018/156574904938542_zh-CN.png)

![执行Pipeline1](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123018/156574904938543_zh-CN.png)

## 访问服务 {#section_dw0_rk3_hd8 .section}

如果部署文件中没有指定namespace，则默认会部署到GitLab命名空间下：

``` {#codeblock_gt1_01h_s3j}
$ kubectl -n gitlab get svc 
NAME        TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
java-demo   LoadBalancer   172.19.9.252   xx.xx.xx.xx   80:32349/TCP   1m
```

浏览器访问xx.xx.xx.xx/demo进行验证。

了解更多容器服务的相关内容，请参见[容器服务](https://www.alibabacloud.com/zh/product/container-service)。

了解更多GitLab CI的相关内容，请参见[GitLab CI](https://docs.gitlab.com/runner/)。

