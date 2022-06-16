[
](#Artifactory性能优化指导-7.服务器文件管理以及最大用户线程相关配置优化)
随着使用人数的和并发量的增加，artifactory的默认配置已经不能满足现在的使用要求，所以需要优化artifactory的配置以满足现在需求

> 所有参数请根据实际情况修改，以下所有推荐配置在32vCPU，64G～128Gmemory 环境下。
> 以下优化参数为V7版本，原文地址：[https://jfrog.com/knowledge-base/how-do-i-tune-artifactory-for-heavy-loads/](https://jfrog.com/knowledge-base/how-do-i-tune-artifactory-for-heavy-loads/)

[
](https://jfrog.com/knowledge-base/how-do-i-tune-artifactory-for-heavy-loads/)
# 1.system.yaml配置优化
> 该配置优化包含jvm 优化，tomcat连接池优化，数据库连接池优化


```yaml
configVersion: 1
shared:
    extraJavaOpts: "-Xms16g -Xmx32g"
    ## Security Configuration
    security: 
    node:
        id: art1

        ## Default: auto resolved by startup script
        ip: 10.194.72.30
        ## Sets this node as part of HA installation
        #haEnabled: true

    ## Database Configuration
    database:
        type: mysql
        ## One of: mysql, oracle, mssql, postgresql, mariadb
        ## Default: Embedded derby
        driver: com.mysql.jdbc.Driver
        url: jdbc:mysql://mysql:3306/artdb?characterEncoding=UTF-8&elideSetAutoCommits=true&useSSL=false
        username: artifactory
        password: 12345678


## ARTIFACTORY TEMPLATE
artifactory:
    database:
        ## Max connections to the database the main connection pool can consume
        maxOpenConnections: 300
        # Max connection to keep idle
        #maxIdleConnections: 10
        # Connection pool manager. Either tomcat-jdbc or hikari
        #poolType: "tomcat-jdbc"
    ## Artifactory Tomcat connector customization on the Artifactory port  
    tomcat:
        ## Artifactory connector settings
        connector:
            maxThreads: 600
            ## An extra configuration to add to the Artifactory connector
            extraConfig: "connectionTimeout=\"12000000\""
## ACCESS TEMPLATE
access:
    ## Database settings for overriding shared.database
    ## Same format as under shared.database. Default embedded database is derby
    database: 
        maxOpenConnections: 300
        # Max connection to keep idle
        #maxIdleConnections: 10
        # Connection pool manager. Either tomcat-jdbc or hikari
        #poolType: "tomcat-jdbc"
    ## Tomcat connector customization on the Access port
    tomcat:
        ## Access connector settings
        connector:
            maxThreads: 150
## METADATA TEMPLATE
metadata:
    ## Database settings for overriding shared.database
    ## Same format as under shared.database. Default embedded database is sqlite
    database:
        ## Max connections to the database the main connection pool can consume
        maxOpenConnections: 300

```
**其他jvm参数：**
Artictory 启动市区非北京时区，如需修改，在jvm 参数里增加 -Duser.timezone=GMT+08

> 重要信息: 修改access 服务的 maxThreads, 需要修改如下配置文件，属性值和maxThreads保持一致
> $JFROG_HOME/artifactory/var/etc/artifactory/artifactory.system.properties artifactory.access.client.max.connections = <VALUE>

# 2.Binarystore.xml （IO配置优化）
     
    提供文件缓存功能（使用SSD硬盘）
       修改$JFROG_HOME/var/etc/artifactory/binarystore.xml
       该 cache-fs 缓存所有的上传/下载请求。这可以提高Artifactory的性能。一般推荐实际总存储的20分之一，例：2TB存储，建议100G ssd缓存空间，可以根据情况适当增加缓存提高命中率
       （参考： https://www.jfrog.com/confluence/display/RTF/Configuring+the+Filestore）
## 2.1 试例一：使用本地磁盘作为持久存储
```xml
<config version="v1">
    <chain template="cache-fs"/>
        <provider id="cache-fs" type="cache-fs">
            <cacheProviderDir>/cache/filestore</cacheProviderDir>
         <maxCacheSize>10000000000</maxCacheSize>
        </provider>       
        <provider id="file-system" type="file-system">
            <fileStoreDir>/opt/jfrog/nfsmount/filestore</fileStoreDir>
        </provider>  
</config>
```
     
## 2.2 试例二：HA环境使用S3存储
```xml
<config version="2">
    <chain template="cluster-s3"/>
    <provider id="cache-fs-eventual-s3" type="cache-fs">
    	<cacheProviderDir>/cache/filestore</cacheProviderDir>
        <maxCacheSize>10000000000</maxCacheSize>
    </provider> 
    <provider id="s3" type="s3">
       <endpoint>http://s3.amazonaws.com</endpoint>
       <identity>[ENTER IDENTITY HERE]</identity>
       <credential>[ENTER CREDENTIALS HERE]</credential>
       <path>[ENTER PATH HERE]</path>
       <bucketName>[ENTER BUCKET NAME HERE]</bucketName>
    </provider>
</config>
```
## 2.3 试例三：单节点挂载NFS，NAS，GFS，fuse等网络存储，优化上传下载速度
PS： 以下配置仅可用于单点挂在NFS等网络存储架构
```xml
<config version="v1">
    <chain template="file-system"/>
    <provider id="file-system" type="file-system">
        <fileStoreDir>/filestore</fileStoreDir>  
    </provider>
    <!-- The eventual provider configuration --> 
    <!-- 加入eventual 配置，文件会先上传到artifactory安装目录下data/eventual/_pre 目录，再拷贝到网络存储，提高上传速度 --> 
    <provider id="eventual" type="eventual">
        <numberOfThreads>10</numberOfThreads>  
        <timeout>180000</timeout>
    </provider>
</config>
```
## 2.4 S3 pool size 和 上传s3线程数量(底层存储使用S3参考优化)
**S3连接数优化**
```xml
<provider id="s3" type="s3">
        <endpoint>commondatastorage.googleapis.com</endpoint>
        <bucketName><BUCKET NAME></bucketName> 
        <identity>XXXXXX</identity>
        <credential>XXXXXXX</credential>
        <property name="httpclient.max-connections" value="100"></property> <!--默认值100， 推荐设置为tomcat 的maxThread数量，详见第二点：maxThread值为2000-->
</provider>
```
注意：s3最大链接数：需要根据集群节点数计算S3服务端支持的最大连接数 (如果一个Artifactory节点允许2k 链接数，并且集群中有2个节点服务器, S3服务端应该允许 2k * 2  = 4k 的连接数)

## 2.5 异步上传到s3线程数优化
```xml
添加配置选项1（模版为s3, 
<chain template="s3"/>
<provider id="eventual" type="eventual"> 
   <numberOfThreads>5</numberOfThreads>  <!--默认为5，推荐100，并发线程将$ARTIFACTORY_HOME/data目录下到文件异步上传到S3-->
   <timeout>180000</timeout>
</provider>
```
PS：待上传到S3持久层相关配置，直接影响本地空间使用率，建议根据磁盘监控调整此值，下列选项配置1和2根据模版的选择来调整：
添加配置选项2 模版为cluster-s3,
```xml
<chain template="cluster-s3"/>
<provider id="eventual-cluster-s3" type="eventual-cluster">
   <maxWorkers>5</maxWorkers> <!--默认为5，推荐100，此提供程序使用的工作线程的数量。这些线程处理针对远程文件存储的所有操作。-->
   <numberOfThreads>5</numberOfThreads>  <!--默认为5，推荐100，并发线程将$ARTIFACTORY_HOME/data目录下到文件异步上传到S3-->
   <timeout>180000</timeout>
</provider>
```

## 2.6 集群remote provider连接数优化

PS：模版为cluster-s3时添加如下配置
```xml
<chain template="cluster-s3"/>
<provider id="remote-s3" type="remote">
    <checkPeriod>15000</checkPeriod>
    <connectionTimeout>5000</connectionTimeout>
    <socketTimeout>15000</socketTimeout><!--当日志出现remote provider read timeout时可以调高该参数，一般出现该问题，仍需整体优化应用性能，CPU，内存，tomcat 线程池等-->
    <maxConnections>200</maxConnections><!--默认值200，建议尽量与tomcat maxThread保持一致-->
    <connectionRetry>2</connectionRetry>
    <zone>remote</zone>
</provider>
```

# 3 外部请求Http Clinet （remote仓库外部请求优化）
 remote 仓库负责代理外网包下载，连接池默认是每一个路由50并发
可以修改该文件提高连接池 `$JFROG_HOME/var/etc/artifactory.system.properties`.
默认值:
```
artifactory.http.client.max.total.connections = 50
artifactory.http.client.max.connections.per.route = 50

优化值范本:
artifactory.http.client.max.total.connections = 200
artifactory.http.client.max.connections.per.route = 80
```

# 4 服务器文件管理以及最大用户线程相关配置优化
        如下配置的环境配置为 32 vcpu，128G memory
              Ulimit –n 500000 
              Ulimit –u 4028  （此值不能低于tocmat maxThread + artifactory.async.corePoolSize，可以适当调高，比如10000）
        如需要永久生效，vi /etc/security/limits.conf
```
artifactory soft nofile 655360
artifactory hard nofile 655360
artifactory soft nproc 4028
artifactory hard nproc 4028
```

