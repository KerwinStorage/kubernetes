## 部署方式一：修改最大内存区域

```
直接起容器报错：max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
#修改宿主机的环境变量
sudo vi /etc/sysctl.conf
vm.max_map_count=655360
sudo sysctl -p

#启动postgres容器
docker run --name pgdb -e POSTGRES_USER=sonar -e POSTGRES_PASSWORD=sonar -p 5433:5432 -v /data/postgresql/data:/var/lib/postgresql/data -d postgres


docker run --name sq --link pgdb -e SONARQUBE_JDBC_USERNAME=sonar -e SONARQUBE_JDBC_PASSWORD=sonar -e SONARQUBE_JDBC_URL=jdbc:postgresql://pgdb:5432/sonar -p 19000:9000 -v /data/sonarqube/data:/opt/sonarqube/data -v /data/sonarqube/extensions:/opt/sonarqube/extensions -v /data/sonarqube/logs:/opt/sonarqube/logs -d sonarqube:8.3.1-community
```

**问题解决：**https://github.com/SonarSource/docker-sonarqube/issues/282

注：**max_map_count**文件包含限制一个进程可以拥有的VMA(虚拟内存区域)的数量。虚拟内存区域是一个连续的虚拟地址空间区域。在进程的生命周期中，每当程序尝试在内存中映射文件，链接到共享内存段，或者分配堆空间的时候，这些区域将被创建。调优这个值将限制进程可拥有VMA的数量。限制一个进程拥有VMA的总数可能导致应用程序出错，因为当进程达到了VMA上线但又只能释放少量的内存给其他的内核进程使用时，操作系统会抛出内存不足的错误。如果你的操作系统在NORMAL区域仅占用少量的内存，那么调低这个值可以帮助释放内存给内核用。

 

## 部署方式二：禁用es

```
docker run --name pgdb --restart always -e POSTGRES_USER=sonar -e POSTGRES_PASSWORD=sonar -p 5433:5432 -v /data/postgresql/data:/var/lib/postgresql/data -d postgres



docker run --name sonarqube  --restart always --link pgdb -e SONARQUBE_JDBC_USERNAME=sonar -e SONARQUBE_JDBC_PASSWORD=sonar -e SONARQUBE_JDBC_URL=jdbc:postgresql://pgdb:5432/sonar -Dsonar.search.javaAdditionalOpts=-Dnode.store.allow_mmapfs=false -p 19000:9000 -v /data/sonarqube/data:/opt/sonarqube/data -v /data/sonarqube/extensions:/opt/sonarqube/extensions -v /data/sonarqube/logs:/opt/sonarqube/logs -d sonarqube:8.3.1-community
```

**注：添加`-Dnode.store.allow_mmapfs=false`到`sonar.search.javaAdditionalOpts`，您将无法使用nmap运行ES。**

**问题解决摘自：**https://community.sonarsource.com/t/unable-to-disable-mmapfs-use-in-elasticsearch/11413

## 配置文件内容

```
sonar.login=admin
sonar.password=admin
sonar.jdbc.url=jdbc:mysql://10.0.0.7:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance&useSSL=false
sonar.web.javaOpts=-server
```

**启动sonarqueb的docker容器**

## 安装插件

**插件地址：**https://binaries.sonarsource.com/

**插件地址：**https://github.com/

**介绍：这里我们演示直接在sonarqube安装也会说明把插件安装到目录下（网络不好，插件直接下载不下来）。**

**8.3.1版本的插件总表：**https://docs.sonarqube.org/latest/instance-administration/plugin-version-matrix/

### 方式一：上传目录

官方插件下载地址：https://binaries.sonarsource.com/Distribution/

8.3.1版本官方文档下载插件：https://docs.sonarqube.org/latest/analysis/languages/csharp/

默认安装插件位置：`/opt/sonarqube/extensions/plugins`。（docker 8.3.1版本目录地址 ，后期可以进入docker容器进行`find / -name "plugins"`查找）这里需要注意sonarqube版本的不同中文插件的安装也会不同，[中文插件地址](https://github.com/SonarQubeCommunity/sonar-l10n-zh)

中文插件版本：

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200603200754104-975268688.png)

 中文包插件安装：

```
wget https://github.com/SonarQubeCommunity/sonar-l10n-zh/releases/download/sonar-l10n-zh-plugin-8.3/sonar-l10n-zh-plugin-8.3.jar
docker cp   sonar-l10n-zh-plugin-8.3.jar  sq:/opt/sonarqube/extensions/plugins/
```

java插件安装：

```
wget https://binaries.sonarsource.com/Distribution/sonar-java-plugin/sonar-java-plugin-6.3.0.21585.jar
docker cp sonar-java-plugin/sonar-java-plugin-6.3.0.21585.jar sq:/opt/sonarqube/extensions/plugins/
```

python插件安装：

```
wget https://binaries.sonarsource.com/Distribution/sonar-python-plugin/sonar-python-plugin-2.9.0.6410.jar
docker cp sonar-python-plugin-2.9.0.6410.jar  sq:/opt/sonarqube/extensions/plugins/
```

C++插件安装：

**c++插件安装地址：**https://github.com/SonarOpenCommunity/sonar-cxx/releases （请先查看兼容表）

```
wget https://binaries.sonarsource.com/CommercialDistribution/sonar-cfamily-plugin/sonar-cfamily-plugin-6.9.0.17076.jar
docker cp sonar-cfamily-plugin-6.9.0.17076.jar  sq:/opt/sonarqube/extensions/plugins/
```

js插件安装：

官方文档：[用于JavaScript](https://docs.sonarqube.org/latest/analysis/languages/javascript/) [用于TypeScript](https://docs.sonarqube.org/latest/analysis/languages/typescript/)

**注：js插件需要安装两个，只安装一个js插件会导致插件不兼容，容器起不来或者服务起不来。**

 

```
wget https://binaries.sonarsource.com/Distribution/sonar-typescript-plugin/sonar-typescript-plugin-2.1.0.4359.jar
docker cp sonar-typescript-plugin-2.1.0.4359.jar sq:/opt/sonarqube/extensions/plugins/


wget https://binaries.sonarsource.com/Distribution/sonar-javascript-plugin/sonar-javascript-plugin-6.2.1.12157.jar
docker cp sonar-javascript-plugin-6.2.1.12157.jar sq:/opt/sonarqube/extensions/plugins/
```

### 方式二：直接安装插件

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200603201712441-1244843260.png)

 

 **结语：安装插件的方式也就这两种，如果网络可以，直接安装，网络差可以选择直接上传到目录。**