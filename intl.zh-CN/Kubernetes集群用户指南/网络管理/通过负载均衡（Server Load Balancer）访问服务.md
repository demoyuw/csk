# 通过负载均衡（Server Load Balancer）访问服务 {#task_1425948 .task}

您可以使用阿里云负载均衡来访问服务。

如果您的集群的cloud-controller-manager版本大于等于v1.9.3，对于指定已有SLB，系统默认不再为该SLB处理监听，用户可以通过设置`service.beta.kubernetes.io/alibaba-cloud-loadbalancer-force-override-listeners: "true"`参数来显示启用监听配置，或者手动配置该SLB的监听规则。

执行以下命令，可查看cloud-controller-manager的版本。

``` {#codeblock_agw_mjd_s5c}
root@master # kubectl get pod -n kube-system -o yaml|grep image:|grep cloud-con|uniq

  image: registry-vpc.cn-hangzhou.aliyuncs.com/acs/cloud-controller-manager-amd64:v1.9.3
```

## 注意事项 {#section_0iv_2kv_rv0 .section}

-   Cloud Controller Manager（简称CCM）会为`Type=LoadBalancer`类型的Service创建或配置阿里云负载均衡（SLB），包含**SLB**、**监听**、**虚拟服务器组**等资源。
-   对于非LoadBalancer类型的service则不会为其配置负载均衡，这包含如下场景：当用户将`Type=LoadBalancer`的service变更为`Type!=LoadBalancer`时，CCM也会删除其原先为该Service创建的SLB（用户通过`service.beta.kubernetes.io/alibaba-cloud-loadbalancer-id`指定的已有SLB除外）。
-   自动刷新配置：CCM使用声明式API，会在一定条件下自动根据service的配置刷新阿里云负载均衡配置，所有用户自行在SLB控制台上修改的配置均存在被覆盖的风险（使用已有SLB同时不覆盖监听的场景除外），因此不能在SLB控制台手动修改Kubernetes创建并维护的SLB的任何配置，否则有配置丢失的风险。
-   同时支持为serivce指定一个已有的负载均衡，或者让CCM自行创建新的负载均衡。但两种方式在SLB的管理方面存在一些差异：

    指定已有SLB

    -   需要为Service设置`service.beta.kubernetes.io/alibaba-cloud-loadbalancer-id` annotation。
    -   SLB配置：此时CCM会使用该SLB做为Service的SLB，并根据其他annotation配置SLB，并且自动的为SLB创建多个虚拟服务器组（当集群节点变化的时候，也会同步更新虚拟服务器组里面的节点）。
    -   监听配置：是否配置监听取决于`service.beta.kubernetes.io/alibaba-cloud-loadbalancer-force-override-listeners:`是否设置为true。 如果设置为false，那么CCM不会为SLB管理任何监听配置。如果设置为true，那么CCM会尝试为SLB更新监听，此时CCM会根据监听名称判断SLB上的监听是否为k8s维护的监听（名字的格式为k8s/Port/ServiceName/Namespace/ClusterID），若Service声明的监听与用户自己管理的监听端口冲突，那么CCM会报错。
    -   SLB的删除： 当Service删除的时候CCM不会删除用户通过id指定的已有SLB。
    CCM管理的SLB

    -   CCM会根据service的配置自动的创建配置**SLB**、**监听**、**虚拟服务器组**等资源，所有资源归CCM管理，因此用户不得手动在SLB控制台更改以上资源的配置，否则CCM在下次Reconcile的时候将配置刷回service所声明的配置，造成非用户预期的结果。
    -   SLB的删除：当Service删除的时候CCM会删除该SLB。
-   后端服务器更新
    -   CCM会自动的为该Service对应的SLB刷新后端虚拟服务器组。当Service对应的后端Endpoint发生变化的时候或者集群节点变化的时候都会自动的更新SLB的后端Server。
    -   `spec.externalTrafficPolicy = Cluster`模式的Service，CCM默认会将所有节点挂载到SLB的后端（使用BackendLabel标签配置后端的除外）。由于SLB限制了每个ECS上能够attach的SLB的个数（quota），因此这种方式会快速的消耗该quota,当quota耗尽后，会造成Service Reconcile失败。解决的办法，可以使用Local模式的Service。
    -   `spec.externalTrafficPolicy = Local`模式的Service，CCM默认只会讲Service对应的Pod所在的节点加入到SLB后端。这会明显降低quota的消耗速度。同时支持四层源IP保留。
    -   任何情况下CCM不会将Master节点作为SLB的后端。
    -   CCM会从SLB后端摘除被kubectl drain/cordon的节点。

## 通过命令行操作 {#section_td0_1qf_nnc .section}

方法一：

1.  通过命令行工具创建一个 Nginx 应用。 

    ``` {#codeblock_92a_nwl_hhe}
    root@master # kubectl run nginx --image=registry.aliyuncs.com/acs/netdia:latest
    root@master # kubectl get po 
    NAME                                   READY     STATUS    RESTARTS   AGE
    nginx-2721357637-dvwq3                 1/1       Running   1          6s
    ```

2.  为 Nginx 应用创建阿里云负载均衡服务，指定 `type=LoadBalancer` 来向外网用户暴露 Nginx 服务。 

    ``` {#codeblock_qp1_5hu_2k3}
    root@master # kubectl expose deployment nginx --port=80 --target-port=80 --type=LoadBalancer
    root@master # kubectl get svc
    NAME                  CLUSTER-IP      EXTERNAL-IP      PORT(S)                        AGE
    nginx                 172.19.XX.XX    101.37.XX.XX     80:31891/TCP                   4s
    ```

3.  在浏览器中访问 `http://101.37.XX.XX`，来访问您的 Nginx 服务。

方法二：

1.  将下面的 yml code 保存到 `nginx-svc.yml`文件中。 

    ``` {#codeblock_o3d_fcy_zf9}
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        run: nignx
      name: nginx-01
      namespace: default
    spec:
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: LoadBalancer
    ```

2.  执行如下命令，创建一个 Nginx 应用。 

    ``` {#codeblock_gek_c9f_gdh}
    kubectl apply -f nginx-svc.yml
    ```

3.  执行如下命令，向外网用户暴露 Nginx 服务。 

    ``` {#codeblock_vrc_n9a_8ua}
    root@master # kubectl get service
    NAME           TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE9d
    ngi-01nx       LoadBalancer   172.19.XX.XX    101.37.XX.XX     80:32325/TCP   3h
    ```

4.  在浏览器中访问 `http://101.37.XX.XX`，来访问您的 Nginx 服务。

## 通过 Kubernetes Dashboard 操作 {#section_9pb_73z_j9u .section}

1.  将下面的 yml code 保存到 `nginx-svc.yml`文件中。 

    ``` {#codeblock_e3u_5pt_uw8}
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        run: nginx
      name: http-svc
      namespace: default
    spec:
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: LoadBalancer
    ```

2.  登录[容器服务管理控制台](https://cs.console.aliyun.com)，单击目标集群右侧的**控制台**，进入 Kubernetes Dashboard 页面。
3.  单击**创建**，开始创建应用。 

    ![创建应用](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16677/15685966199066_zh-CN.png)

4.  单击**使用文件创建**。选择刚才保存的nginx-svc.yml 文件。
5.  单击**上传**。 此时，会创建一个阿里云负载均衡实例指向创建的 Nginx 应用，服务的名称为 `http-svc`。
6.  在 Kubernetes Dashboard 上定位到 default 命名空间，选择服务。 可以看到刚刚创建的 `http-svc` 的 Nginx 服务和机器的负载均衡地址 `http://114.55.XX.XX:80`。

    ![访问服务](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16677/15685966199067_zh-CN.png)

7.  将该地址拷贝到浏览器中即可访问该服务。

## 通过控制台操作 {#section_jcf_02c_7om .section}

1.  登录[容器服务管理控制台](https://cs.console.aliyun.com)。
2.  在 Kubernetes 菜单下，单击左侧导航栏中的**应用** \> **无状态**，进入无状态（Deployment）页面。
3.  选择目标集群和命名空间，单击右上角**使用模板创建**。 

    ![创建应用](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16677/156859661913797_zh-CN.png)

4.  **示例模板**选为自定义，将以下内容复制到**模板**中。 

    ``` {#codeblock_z6q_bix_jre}
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        run: nginx
      name: ngnix
      namespace: default
    spec:
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: LoadBalancer
    ```

5.  单击**创建**。
6.  创建成功，单击**Kubernetes 控制台**前往控制台查看创建进度。 

    ![Kubernetes 控制台](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16677/156859661913798_zh-CN.png)

7.  或单击左侧导航栏**路由与负载均衡** \> **服务**，选择目标集群和命名空间，查看已部署的服务。 

    ![部署服务](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16677/156859661913800_zh-CN.png)


## 更多信息 {#section_jnq_v5w_nla .section}

阿里云负载均衡还支持丰富的配置参数，包含健康检查、收费类型、负载均衡类型等参数。详细信息参见[负载均衡配置参数表](#table_y0y_w2b_ajg)。

## 注释 {#section_75c_ujo_g9c .section}

阿里云可以通过注释`annotations`的形式支持丰富的负载均衡功能。

-   创建一个公网类型的负载均衡

    ``` {#codeblock_06o_6ob_zzd}
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   创建一个私网类型的负载均衡

    ``` {#codeblock_hhp_pp9_mw4}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-address-type: "intranet"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   创建HTTP类型的负载均衡

    ``` {#codeblock_a4r_eds_ecj}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-protocol-port: "http:80"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   创建HTTPS类型的负载均衡

    需要先在阿里云控制台上创建一个证书并记录 cert-id，然后使用如下 annotation 创建一个 HTTPS 类型的 SLB。

    ``` {#codeblock_7te_4ag_g7n}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-protocol-port: "https:443"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-cert-id: "${YOUR_CERT_ID}"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 443
        protocol: TCP
        targetPort: 443
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   限制负载均衡的带宽

    ``` {#codeblock_zqm_ct2_x65}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-charge-type: "paybybandwidth"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-bandwidth: "100"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 443
        protocol: TCP
        targetPort: 443
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   指定负载均衡规格

    负载均衡规格可参见[CreateLoadBalancer](../../../../../intl.zh-CN/API参考/负载均衡实例/CreateLoadBalancer.md#)。

    ``` {#codeblock_e8t_74y_rtz}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-spec: "slb.s1.small"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 443
        protocol: TCP
        targetPort: 443
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   使用已有的负载均衡

    -   默认情况下，使用已有的负载均衡实例，不会覆盖监听，如要强制覆盖已有监听，请配置`service.beta.kubernetes.io/alibaba-cloud-loadbalancer-force-override-listeners`为ture 

        **说明：** 复用已有的负载均衡默认不覆盖已有监听，出于以下两点原因：

        -   如果已有负载均衡的监听上绑定了业务，强制覆盖会引发业务中断
        -   由于CCM目前支持的后端配置有限，无法处理一些复杂配置。如果有复杂的后端配置需求，可以通过手动方式自行配置。
        如存在以上两种情况不建议强制覆盖监听，如果已有负载均衡的监听端口不再使用，则可以强制覆盖。

    -   使用已有的负载均衡暂不支持添加额外标签（`annotation: service.beta.kubernetes.io/alibaba-cloud-loadbalancer-additional-resource-tags`）
    ``` {#codeblock_03f_moe_yx1}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-id: "${YOUR_LOADBALACER_ID}"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 443
        protocol: TCP
        targetPort: 443
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   使用已有的负载均衡，并强制覆盖已有监听

    强制覆盖已有监听，如果监听端口冲突，则会删除已有监听。

    ``` {#codeblock_z43_dsq_fv1}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-id: "${YOUR_LOADBALACER_ID}"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-force-override-listeners: "true"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 443
        protocol: TCP
        targetPort: 443
      selector:
        run: nginx
      type: LoadBalancere: LoadBalancer
    ```

-   使用指定Label的worker节点作为后端服务器

    多个Label以逗号分隔。例如："k1=v1,k2=v2"。多个label之间是and的关系。

    ``` {#codeblock_io4_ggi_prz}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-backend-label: "failure-domain.beta.kubernetes.io/zone=ap-southeast-5a"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 443
        protocol: TCP
        targetPort: 443
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   为TCP类型的负载均衡配置会话保持保持时间

    -   参数`service.beta.kubernetes.io/alibaba-cloud-loadbalancer-persistence-time`仅对TCP协议的监听生效。
    -   如果负载均衡实例配置了多个TCP协议的监听端口，则默认将该配置应用到所有TCP协议的监听端口。
    ``` {#codeblock_7d4_d5o_2hy}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-persistence-timeout: "1800"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 443
        protocol: TCP
        targetPort: 443
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   为HTTP&HTTPS协议的负载均衡配置会话保持（insert cookie）

    -   仅支持HTTP及HTTPS协议的负载均衡实例。
    -   如果配置了多个HTTP或者HTTPS的监听端口，该会话保持默认应用到所有HTTP和HTTPS监听端口。
    -   配置insert cookie，以下四项annotation必选。
    ``` {#codeblock_q9z_594_lce}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-sticky-session: "on"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-sticky-session-type: "insert"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-cookie-timeout: "1800"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-protocol-port: "http:80"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   为HTTP&HTTPS协议的负载均衡配置会话保持（server cookie）

    -   仅支持HTTP及HTTPS协议的负载均衡实例。
    -   如果配置了多个HTTP或者HTTPS的监听端口，该会话保持默认应用到所有HTTP和HTTPS监听端口。
    -   配置server cookie，以下四项annotation必选。
    -   cookie名称（service.beta.kubernetes.io/alibaba-cloud-loadbalancer-cookie）只能包含字母、数字、‘\_’和‘-’。
    ``` {#codeblock_66o_vyu_lgv}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-sticky-session: "on"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-sticky-session-type: "server"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-cookie: "${YOUR_COOKIE}"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-protocol-port: "http:80"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   创建负载均衡时，指定主备可用区

    -   某些region的负载均衡不支持主备可用区，例如ap-southeast-5。
    -   一旦创建，主备可用区不支持修改。
    ``` {#codeblock_124_g1h_4j8}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-master-zoneid: "ap-southeast-5a"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-slave-zoneid: "ap-southeast-5a"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   使用Pod所在的节点作为后端服务器

    默认`externalTrafficPolicy`为Cluster模式，会将集群中所有节点挂载到后端服务器。Local模式仅将Pod所在节点作为后端服务器。

    ``` {#codeblock_5w9_3i9_7a0}
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx
      namespace: default
    spec:
      externalTrafficPolicy: Local
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   创建私有网络类型（VPC）的负载均衡

    -   创建私有网络类型的负载均衡，以下两个annotation必选。
    -   私网负载均衡支持专有网络（VPC）\)和经典网络（Classic），两者区别参见[实例概述](../../../../../intl.zh-CN/实例/实例概述.md#)。
    ``` {#codeblock_eyl_08w_ay7}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-address-type: "intranet"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-network-type: "vpc"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 443
        protocol: TCP
        targetPort: 443
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   创建按流量付费的负载均衡

    -   仅支持公网类型的负载均衡实例
    -   以下两项annotation必选
    ``` {#codeblock_eqb_8pp_xiv}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-bandwidth: "45"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-charge-type: "paybybandwidth"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 443
        protocol: TCP
        targetPort: 443
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   创建带健康检查的负载均衡
    -   设置TCP类型的健康检查

        -   TCP端口默认开启健康检查，且不支持修改，即`service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-flag annotation`无效。
        -   设置TCP类型的健康检查，以下所有annotation必选。
        ``` {#codeblock_49r_i4b_hpu}
        apiVersion: v1
        kind: Service
        metadata:
          annotations:
            service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-type: "tcp"
            service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-connect-timeout: "8"
            service.beta.kubernetes.io/alibaba-cloud-loadbalancer-healthy-threshold: "4"
            service.beta.kubernetes.io/alibaba-cloud-loadbalancer-unhealthy-threshold: "4"
            service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-interval: "3"
          name: nginx
          namespace: default
        spec:
          ports:
          - port: 80
            protocol: TCP
            targetPort: 80
          selector:
            run: nginx
          type: LoadBalancer
        ```

    -   设置HTTP类型的健康检查

        设置HTTP类型的健康检查，以下所有的annotation必选。

        ``` {#codeblock_z1w_ob3_lm0}
        apiVersion: v1
        kind: Service
        metadata:
          annotations:
            service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-flag: "on"
            service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-type: "http"
            service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-uri: "/test/index.html"
            service.beta.kubernetes.io/alibaba-cloud-loadbalancer-healthy-threshold: "4"
            service.beta.kubernetes.io/alibaba-cloud-loadbalancer-unhealthy-threshold: "4"
            service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-timeout: "10"
            service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-interval: "3"
            service.beta.kubernetes.io/alibaba-cloud-loadbalancer-protocol-port: "http:80"
          name: nginx
          namespace: default
        spec:
          ports:
          - port: 80
            protocol: TCP
            targetPort: 80
          selector:
            run: nginx
          type: LoadBalancer
        ```

-   为负载均衡设置调度算法

    -   wrr（默认值）：权重值越高的后端服务器，被轮询到的次数（概率）也越高。
    -   wlc：除了根据每台后端服务器设定的权重值来进行轮询，同时还考虑后端服务器的实际负载（即连接数）。当权重值相同时，当前连接数越小的后端服务器被轮询到的次数（概率）也越高。
    -   rr：按照访问顺序依次将外部请求依序分发到后端服务器。
    ``` {#codeblock_yrx_skr_fez}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-scheduler: "wlc"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 443
        protocol: TCP
        targetPort: 443
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   创建带有访问控制的负载均衡

    -   需要先在阿里云控制台上创建一个访问控制并记录acl-id，然后使用如下 annotation 创建一个带有访问控制的负载均衡实例。
    -   白名单适合只允许特定IP访问的场景，black黑名单适用于只限制某些特定IP访问的场景。
    -   创建带有访问控制的负载均衡，以下三项annotation必选。
    ``` {#codeblock_u1l_mvx_vw3}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-acl-status: "on"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-acl-id: "${YOUR_ACL_ID}"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-acl-type: "white"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 443
        protocol: TCP
        targetPort: 443
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   为负载均衡指定虚拟交换机

    -   通过阿里云专有网络控制台查询交换机ID，然后使用如下的annotation为负载均衡实例指定虚拟交换机。
    -   为负载均衡指定虚拟交换机，以下两项annotation必选。
    ``` {#codeblock_vlr_0jn_7kn}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
       service.beta.kubernetes.io/alibaba-cloud-loadbalancer-address-type: "intranet"
       service.beta.kubernetes.io/alibaba-cloud-loadbalancer-vswitch-id: "${YOUR_VSWITCH_ID}"
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 443
        protocol: TCP
        targetPort: 443
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   为负载均衡指定转发端口

    -   端口转发是指将http端口的请求转发到https端口上。
    -   设置端口转发需要先在阿里云控制台上创建一个证书并记录cert-id。
    -   如需设置端口转发，以下三项annotation必选。
    ``` {#codeblock_cdm_1az_99w}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-protocol-port: "https:443,http:80"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-cert-id: "${YOUR_CERT_ID}"
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-forward-port: "80:443"
      name: nginx
      namespace: default
    spec:
      ports:
      - name: https
        port: 443
        protocol: TCP
        targetPort: 443
      - name: http
        port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: LoadBalancer
    ```

-   为负载均衡添加额外标签

    多个tag以逗号分隔，例如："k1=v1,k2=v2"。

    ``` {#codeblock_ru0_j41_tcf}
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/alibaba-cloud-loadbalancer-additional-resource-tags: "Key1=Value1,Key2=Value2" 
      name: nginx
      namespace: default
    spec:
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: LoadBalancer
    ```


**说明：** 

-   注释的内容是区分大小写的。
-   自2019年9月11日起，annotation字段alicloud更新为alibaba-cloud。

    例如：

    更新前：`service.beta.kubernetes.io/alicloud-loadbalancer-id`

    更新后：`service.beta.kubernetes.io/alibaba-cloud-loadbalancer-id`

    系统将继续兼容alicloud的写法，用户无需做任何修改，敬请注意。


|注释|类型|描述|默认值|
|--|--|--|---|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-protocol-port|string|多个值之间由逗号分隔，例如：https:443,http:80|无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-address-type|string|取值可以是internet或者intranet|internet|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-slb-network-type|string|负载均衡的网络类型，取值可以是classic或者vpc 取值为vpc时，需设置`service.beta.kubernetes.io/alibaba-cloud-loadbalancer-address-type`为intranet。

 |classic|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-charge-type|string|取值可以是paybytraffic或者paybybandwidth|paybytraffic|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-id|string|负载均衡实例的 ID。通过 service.beta.kubernetes.io/alibaba-cloud-loadbalancer-id指定您已有的SLB，默认情况下，使用已有的负载均衡实例，不会覆盖监听，如要强制覆盖已有监听，请配置service.beta.kubernetes.io/alibaba-cloud-loadbalancer-force-override-listeners为true。|无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-backend-label|string|通过 label 指定 SLB 后端挂载哪些worker节点。|无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-spec|string|负载均衡实例的规格。可参见：[CreateLoadBalancer](../../../../../intl.zh-CN/API参考/负载均衡实例/CreateLoadBalancer.md#)|无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-persistence-timeout|string|会话保持时间。 仅针对TCP协议的监听，取值：0-3600（秒）

 默认情况下，取值为0，会话保持关闭。

 可参见：[CreateLoadBalancerTCPListener](../../../../../intl.zh-CN/API参考/TCP监听/CreateLoadBalancerTCPListener.md#)

 |0|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-sticky-session|string|是否开启会话保持。取值：on | off **说明：** 仅对HTTP和HTTPS协议的监听生效。

 可参见：[CreateLoadBalancerHTTPListener](../../../../../intl.zh-CN/API参考/HTTP监听/CreateLoadBalancerHTTPListener.md#)和[CreateLoadBalancerHTTPSListener](../../../../../intl.zh-CN/API参考/HTTPS监听/CreateLoadBalancerHTTPSListener.md#)

 |off|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-sticky-session-type|string|cookie的处理方式。取值： -   insert：植入Cookie。
-   server：重写Cookie。

 **说明：** 

-   仅对HTTP和HTTPS协议的监听生效。
-   当service.beta.kubernetes.io/alibaba-cloud-loadbalancer-sticky-session取值为on时，该参数必选。

可参见：[CreateLoadBalancerHTTPListener](../../../../../intl.zh-CN/API参考/HTTP监听/CreateLoadBalancerHTTPListener.md#)和[CreateLoadBalancerHTTPSListener](../../../../../intl.zh-CN/API参考/HTTPS监听/CreateLoadBalancerHTTPSListener.md#)

 |无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-cookie-timeout|string|Cookie超时时间。取值：1-86400（秒） **说明：** 当service.beta.kubernetes.io/alibaba-cloud-loadbalancer-sticky-session为on且service.beta.kubernetes.io/alibaba-cloud-loadbalancer-sticky-session-type为insert时，该参数必选。

 可参见：[CreateLoadBalancerHTTPListener](../../../../../intl.zh-CN/API参考/HTTP监听/CreateLoadBalancerHTTPListener.md#)和[CreateLoadBalancerHTTPSListener](../../../../../intl.zh-CN/API参考/HTTPS监听/CreateLoadBalancerHTTPSListener.md#)

 |无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-cookie|string|服务器上配置的Cookie名称。 长度为1-200个字符，只能包含ASCII英文字母和数字字符，不能包含逗号、分号或空格，也不能以$开头。

 **说明：** 

当service.beta.kubernetes.io/alibaba-cloud-loadbalancer-sticky-session为on且service.beta.kubernetes.io/alibaba-cloud-loadbalancer-sticky-session-type为server时，该参数必选。

 可参见：[CreateLoadBalancerHTTPListener](../../../../../intl.zh-CN/API参考/HTTP监听/CreateLoadBalancerHTTPListener.md#)和[CreateLoadBalancerHTTPSListener](../../../../../intl.zh-CN/API参考/HTTPS监听/CreateLoadBalancerHTTPSListener.md#)

 |无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-master-zoneid|string|主后端服务器的可用区ID。|无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-slave-zoneid|string|备后端服务器的可用区ID。|无|
|externalTrafficPolicy|string|哪些节点可以作为后端服务器，取值： -   Cluster：使用所有后端节点作为后端服务器。
-   Local：使用Pod所在节点作为后端服务器。

 |Cluster|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-force-override-listeners|string|绑定已有负载均衡时，是否强制覆盖该SLB的监听。|false：不覆盖|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-bandwidth|string|负载均衡的带宽，仅适用于公网类型的负载均衡。|50|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-cert-id|string|阿里云上的证书 ID。您需要先上传证书|无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-flag|string|取值是on | off -   TCP监听默认为on且不可更改。
-   HTTP监听默认为off。

 |默认为off。TCP 不需要改参数。因为 TCP 默认打开健康检查，用户不可设置。|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-type|string|健康检查类型，取值：tcp | http。 可参见：[CreateLoadBalancerTCPListener](../../../../../intl.zh-CN/API参考/TCP监听/CreateLoadBalancerTCPListener.md#)

 |tcp|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-uri|string|用于健康检查的URI。 **说明：** 当健康检查类型为TCP模式时，无需配置该参数。

 可参见：[CreateLoadBalancerTCPListener](../../../../../intl.zh-CN/API参考/TCP监听/CreateLoadBalancerTCPListener.md#)

 |无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-connect-port|string|健康检查使用的端口。取值： -   -520：默认使用监听配置的后端端口。
-   1-65535：健康检查的后端服务器的端口。

 可参见：[CreateLoadBalancerTCPListener](../../../../../intl.zh-CN/API参考/TCP监听/CreateLoadBalancerTCPListener.md#)

 |无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-healthy-threshold|string|健康检查连续成功多少次后，将后端服务器的健康检查状态由fail判定为success。 取值：2-10

 可参见：[CreateLoadBalancerTCPListener](../../../../../intl.zh-CN/API参考/TCP监听/CreateLoadBalancerTCPListener.md#)

 |3|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-unhealthy-threshold|string|健康检查连续失败多少次后，将后端服务器的健康检查状态由success判定为fail。取值： 2-10

 可参见：[CreateLoadBalancerTCPListener](../../../../../intl.zh-CN/API参考/TCP监听/CreateLoadBalancerTCPListener.md#)

 |3|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-interval|string|健康检查的时间间隔。 取值：1-50（秒）

 可参见：[CreateLoadBalancerTCPListener](../../../../../intl.zh-CN/API参考/TCP监听/CreateLoadBalancerTCPListener.md#)

 |2|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-connect-timeout|string|接收来自运行状况检查的响应需要等待的时间，适用于TCP模式。如果后端ECS在指定的时间内没有正确响应，则判定为健康检查失败。 取值：1-300（秒）

 **说明：** 如果service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-connect-timeout的值小于service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-interval的值，则service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-connect-timeout无效，超时时间为service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-interval的值。

 可参见：[CreateLoadBalancerTCPListener](../../../../../intl.zh-CN/API参考/TCP监听/CreateLoadBalancerTCPListener.md#)

 |5|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-timeout|string|接收来自运行状况检查的响应需要等待的时间，适用于HTTP模式。如果后端ECS在指定的时间内没有正确响应，则判定为健康检查失败。 取值：1-300（秒）

 **说明：** 如果 service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-timeout的值小于service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-interval的值，则 service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-timeout无效，超时时间为 service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-interval的值。

 可参见：[CreateLoadBalancerTCPListener](../../../../../intl.zh-CN/API参考/TCP监听/CreateLoadBalancerTCPListener.md#)

 |5|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-domain|string|用于健康检查的域名。 -   $\_ip：后端服务器的私网IP。当指定了IP或该参数未指定时，负载均衡会使用各后端服务器的私网IP当做健康检查使用的域名。
-   domain：域名长度为1-80，只能包含字母、数字、点号（.）和连字符（-）。

 |无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-health-check-httpcode|string|健康检查正常的HTTP状态码，多个状态码用逗号（,）分割。取值： -   http\_2xx
-   http\_3xx
-   http\_4xx
-   http\_5xx

 默认值为http\_2xx。|http\_2xx|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-scheduler|string|调度算法。取值wrr | wlc| rr。 -   wrr：默认取值，权重值越高的后端服务器，被轮询到的次数（概率）也越高。
-   wlc：除了根据每台后端服务器设定的权重值来进行轮询，同时还考虑后端服务器的实际负载（即连接数）。当权重值相同时，当前连接数越小的后端服务器被轮询到的次数（概率）也越高。
-   rr：按照访问顺序依次将外部请求依序分发到后端服务器。

 |wrr|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-acl-status|string|是否开启访问控制功能。取值：on | off|off|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-acl-id|string|监听绑定的访问策略组ID。当AclStatus参数的值为on时，该参数必选。|无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-acl-type|string|访问控制类型。 取值：white | black。

 -   white：仅转发来自所选访问控制策略组中设置的IP地址或地址段的请求，白名单适用于应用只允许特定IP访问的场景。设置白名单存在一定业务风险。一旦设名单，就只有白名单中的IP可以访问负载均衡监听。如果开启了白名单访问，但访问策略组中没有添加任何IP，则负载均衡监听会转发全部请求。
-   black： 来自所选访问控制策略组中设置的IP地址或地址段的所有请求都不会转发，黑名单适用于应用只限制某些特定IP访问的场景。如果开启了黑名单访问，但访问策略组中没有添加任何IP，则负载均衡监听会转发全部请求。当AclStatus参数的值为on时，该参数必选。

 |无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-vswitch-id|string|负载均衡实例所属的VSwitch ID。设置该参数时需同时设置addresstype为intranet。|无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-forward-port|string|将HTTP请求转发至HTTPS指定端口。取值如80:443|无|
|service.beta.kubernetes.io/alibaba-cloud-loadbalancer-additional-resource-tags|string|需要添加的Tag列表，多个标签用逗号分隔。例如："k1=v1,k2=v2"|无|

