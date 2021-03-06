# 通过 Gateway 访问 {#concept_umc_k3l_z2b .concept}

本节介绍使用 HAProxy 反向代理实现通过 Gateway 节点访问 Presto 服务的方法。该方法也很容扩展到其他组件，如 Impala 等。

Gateway 是与 EMR 集群处于同一个内网中的 ECS 服务器，可以使用 Gateway 实现负载均衡和安全隔离。您可以通过**控制台页面** \> **概览** \> **创建 Gateway**，也可以通过**控制台页面** \> **集群管理** \> **创建 Gateway**来创建对应集群的 Gateway 节点。

**说明：** Gateway 节点默认已经安装了 HAProxy 服务，但没有启动。

## 普通集群 {#section_ec3_53l_z2b .section}

普通集群配置 Gateway 代理比较简单，只需要配置 HAProxy 反向代理，对 EMR 集群上Header 节点的 Presto Coodrinator 的 9090 端口实现反向代理即可。配置步骤如下：

1.  配置 HAProxy

    通过 SSH 登入Gateway节点，修改 HAProxy 的配置文件/etc/haproxy/haproxy.cfg。添加如下内容：

    ```
    #---------------------------------------------------------------------
    # Global settings
    #---------------------------------------------------------------------
    global
    ......
    ## 配置代理，将Gateway的9090端口映射到
    ## emr-header-1.cluster-xxxx的9090端口
    listen prestojdbc :9090
        mode tcp
        option tcplog
        balance source
        server presto-coodinator-1 emr-header-1.cluster-xxxx:9090
    ```

2.  保存退出，使用如下命令重启 HAProxy 服务：

    ```
    $> service haproxy restart
    ```

3.  配置安全组

    需要配置的规则如下：

    |方向|配置规则|说明|
    |:-|:---|:-|
    |公网入|自定义 TCP，开放 9090 端口|该端口用于 HAProxy 代理 Header 节点 Coodinator 端口|


至此就可以通过 ECS 控制台删除 Header 节点的公网 IP，在自己的客户机上通过 Gateway 访问 Presto 服务了。

-   命令行使用 Presto 的示例请参见：[命令行工具](cn.zh-CN/开源组件介绍/Presto/快速入门/命令行工具.md#)
-   JDBC 访问 Presto 的示例请参见：[使用 JDBC](cn.zh-CN/开源组件介绍/Presto/快速入门/使用 JDBC.md#)

## 高安全集群 {#section_kg2_sjl_z2b .section}

EMR 高安全集群中的 Presto 服务使用 Kerberos 服务进行认证，其中 Kerberos KDC 服务位于 emr-header-1 上，端口为 88，同时支持 TCP/UDP 协议。 使用 Gateway 访问高安全集群中的 Presto 服务，需要同时对 Presto Coodinator 服务端口和 Kerberos KDC 实现代理。另外，EMR Presto Coodinator 集群默认使用 keystore 配置的 CN 为 emr-header-1，只能内网使用，因此需要重新生成 CN=emr-header-1.cluster-xxx 的 keystore。

-   HTTPS 认证相关
    1.  创建服务端 CN=emr-header-1.cluster-xxx的keystore：

        ```
        [root@emr-header-1 presto-conf]# keytool -genkey -dname "CN=emr-header-1.cluster-xxx,OU=Alibaba,O=Alibaba,L=HZ, ST=zhejiang, C=CN" -alias server -keyalg RSA -keystore keystore -keypass 81ba14ce6084 -storepass 81ba14ce6084 -validity 36500
        Warning:
        JKS 密钥库使用专用格式。建议使用 "keytool -importkeystore -srckeystore keystore -destkeystore keystore -deststoretype pkcs12" 迁移到行业标准格式 PKCS12。
        ```

    2.  导出证书：

        ```
        [root@emr-header-1 presto-conf]# keytool -export -alias server -file server.cer -keystore keystore -storepass 81ba14ce6084
        存储在文件 <server.cer> 中的证书
        Warning:
        JKS 密钥库使用专用格式。建议使用 "keytool -importkeystore -srckeystore keystore -destkeystore keystore -deststoretype pkcs12" 迁移到行业标准格式 PKCS12。
        ```

    3.  制作客户端 keystore：

        ```
        [root@emr-header-1 presto-conf]# keytool -genkey -dname "CN=myhost,OU=Alibaba,O=Alibaba,L=HZ, ST=zhejiang, C=CN" -alias client -keyalg RSA -keystore client.keystore -keypass 123456 -storepass 123456 -validity 36500
        Warning:
        JKS 密钥库使用专用格式。建议使用 "keytool -importkeystore -srckeystore client.keystore -destkeystore client.keystore -deststoretype pkcs12" 迁移到行业标准格式 PKCS12。
        ```

    4.  将证书导入到客户端 keystore 中：

        ```
        [root@emr-header-1 presto-conf]# keytool -import -alias server -keystore client.keystore -file server.cer -storepass 123456
        所有者: CN=emr-header-2.cluster-xxx, OU=Alibaba, O=Alibaba, L=HZ, ST=zhejiang, C=CN
        发布者: CN=emr-header-2.cluster-xxx, OU=Alibaba, O=Alibaba, L=HZ, ST=zhejiang, C=CN
        序列号: 4247108
        有效期为 Thu Mar 01 09:11:31 CST 2018 至 Sat Feb 05 09:11:31 CST 2118
        证书指纹:
             MD5:  75:2A:AA:40:01:5B:3F:86:8F:9A:DB:B1:85:BD:44:8A
             SHA1: C7:25:B9:AD:5F:FE:FC:05:8E:A0:24:4A:1C:AA:6A:8D:6C:39:28:16
             SHA256: DB:86:69:65:73:D5:C6:E2:98:7C:4A:3B:31:EF:70:80:F0:3C:3B:0C:14:94:37:9F:9C:22:47:EA:7E:1E:DE:8C
        签名算法名称: SHA256withRSA
        主体公共密钥算法: 2048 位 RSA 密钥
        版本: 3
        扩展:
        #1: ObjectId: 2.5.29.14 Criticality=false
        SubjectKeyIdentifier [
        KeyIdentifier [
        0000: 45 1D A9 C7 D5 4E BB CF   BD CE B4 5E E2 16 FB 2F  E....N.....^.../
        0010: E9 5D 4A B6                                        .]J.
        ]
        ]
        是否信任此证书? [否]:  是
        证书已添加到密钥库中
        Warning:
        JKS 密钥库使用专用格式。建议使用 "keytool -importkeystore -srckeystore client.keystore -destkeystore client.keystore -deststoretype pkcs12" 迁移到行业标准格式 PKCS12。
        ```

    5.  将生成的文件拷贝到客户端：

        ```
        $> scp root@xxx.xxx.xxx.xxx:/etc/ecm/presto-conf/client.keystore ./
        ```

-   Kerberos 认证相关
    1.  添加客户端用户 principal：

        ```
        [root@emr-header-1 presto-conf]# sh /usr/lib/has-current/bin/hadmin-local.sh /etc/ecm/has-conf -k /etc/ecm/has-conf/admin.keytab
        [INFO] conf_dir=/etc/ecm/has-conf
        Debug is  true storeKey true useTicketCache false useKeyTab true doNotPrompt true ticketCache is null isInitiator true KeyTab is /etc/ecm/has-conf/admin.keytab refreshKrb5Config is true principal is kadmin/EMR.xxx.COM@EMR.xxx.COM tryFirstPass is false useFirstPass is false storePass is false clearPass is false
        Refreshing Kerberos configuration
        principal is kadmin/EMR.xxx.COM@EMR.xxx.COM
        Will use keytab
        Commit Succeeded
        Login successful for user: kadmin/EMR.xxx.COM@EMR.xxx.COM
        enter "cmd" to see legal commands.
        HadminLocalTool.local: addprinc -pw 123456 clientuser
        Success to add principal :clientuser
        HadminLocalTool.local: ktadd -k /root/clientuser.keytab clientuser
        Principal export to keytab file : /root/clientuser.keytab successful .
        HadminLocalTool.local: exit
        ```

    2.  将生成的文件拷贝到客户端：

        ```
        $> scp root@xxx.xxx.xxx.xxx:/root/clientuser.keytab ./
        $> scp root@xxx.xxx.xxx.xxx:/etc/krb5.conf ./
        ```

    3.  修改拷贝到客户端的 krb5.conf 文件，修改如下两处：

        ```
        [libdefaults]
            kdc_realm = EMR.xxx.COM
            default_realm = EMR.xxx.COM
            # 改成1，使客户端使用TCP协议与KDC通信（因为HAProxy不支持UDP协议）
            udp_preference_limit = 1 
            kdc_tcp_port = 88
            kdc_udp_port = 88
            dns_lookup_kdc = false
        [realms]
            EMR.xxx.COM = {
                # 设置为Gateway的外网IP
                kdc = xxx.xxx.xxx.xxx:88
            }
        ```

    4.  修改客户端主机的 hosts 文件，添加如下内容：

        ```
        #  gateway ip
        xxx.xxx.xxx.xxx emr-header-1.cluster-xxx
        ```

-   配置 Gateway HAProxy

    1.  通过 SSH 登入到 Gateway 节点，修改/etc/haproxy/haproxy.cfg。添加如下内容：

        ```
        #---------------------------------------------------------------------
        # Global settings
        #---------------------------------------------------------------------
        global
        ......
        listen prestojdbc :7778
            mode tcp
            option tcplog
            balance source
            server presto-coodinator-1 emr-header-1.cluster-xxx:7778
        listen kdc :88
            mode tcp
            option tcplog
            balance source
            server emr-kdc emr-header-1:88
        ```

    2.  保存退出，使用如下命令重启HAProxy服务：

        ```
        $> service haproxy restart
        ```

    3.  配置安全组规则

        需要配置的规则如下：

        |方向|配置规则|说明|
        |:-|:---|:-|
        |公网入|自定义 UDP，开放 88 端口|该端口用户 HAProxy 代理 Header 节点上的 KDC|
        |公网入|自定义 TCP，开放 88 端口|该端口用户 HAProxy 代理Header节点上的 KDC|
        |公网入|自定义 TCP，开放 7778 端口|该端口用于 HAProxy 代理 Header 节点 Coodinator 端口|

    至此，就可以通过 ECS 控制台删除 Header 节点的公网 IP，在自己的客户机上通过 Gateway 访问 Presto 服务了。

-   使用 JDBC 访问 Presto 示例

    代码如下：

    ```
    try {
        Class.forName("com.facebook.presto.jdbc.PrestoDriver");
    } catch(ClassNotFoundException e) {
        LOG.error("Failed to load presto jdbc driver.", e);
        System.exit(-1);
    }
    Connection connection = null;
    Statement statement = null;
    try {
        String url = "jdbc:presto://emr-header-1.cluster-59824:7778/hive/default";
        Properties properties = new Properties();
        properties.setProperty("user", "hadoop");
        // https相关配置
        properties.setProperty("SSL", "true");
        properties.setProperty("SSLTrustStorePath", "resources/59824/client.keystore");
        properties.setProperty("SSLTrustStorePassword", "123456");
        // Kerberos相关配置
        properties.setProperty("KerberosRemoteServiceName", "presto");
        properties.setProperty("KerberosPrincipal", "clientuser@EMR.59824.COM");
        properties.setProperty("KerberosConfigPath", "resources/59824/krb5.conf");
        properties.setProperty("KerberosKeytabPath", "resources/59824/clientuser.keytab");
        // 创建连接对象
        connection = DriverManager.getConnection(url, properties);
        // 创建Statement对象
        statement = connection.createStatement();
        // 执行查询
        ResultSet rs = statement.executeQuery("select * from table1");
        // 获取结果
        int columnNum = rs.getMetaData().getColumnCount();
        int rowIndex = 0;
        while (rs.next()) {
            rowIndex++;
            for(int i = 1; i <= columnNum; i++) {
                System.out.println("Row " + rowIndex + ", Column " + i + ": " + rs.getString(i));
            }
        }
    } catch(SQLException e) {
        LOG.error("Exception thrown.", e);
    } finally {
        // 销毁Statement对象
        if (statement != null) {
            try {
                statement.close();
                } catch(Throwable t) {
                  // No-ops
                }
        }
       // 关闭连接
       if (connection != null) {
              try {
               connection.close();
           } catch(Throwable t) {
             // No-ops
           }
        }
    }
    ```


## 小结 {#section_m1s_4j5_xgb .section}

本文介绍了使用 HAProxy 反向代理实现通过 Gateway 节点访问 Presto 服务的方法。该方法也很容扩展到其他组件，如 Impala。

