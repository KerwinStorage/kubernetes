## 1. 插件说明

**官方插件地址**：[点击这里](https://plugins.jenkins.io/generic-webhook-trigger/)

**GitHub地址**：[点击这里](https://github.com/jenkinsci/generic-webhook-trigger-plugin)

 jenkins安装完插件会暴露一个公共的APIL：**JENKINS_URL/generic-webhook-trigger/invoke** 当我们成功触发时会获取一些反馈信息，我们可以从这些信息里面利用JSONPath或者XPath提取过滤条件。

> JsonPath为Json文档提供了解析能力，通过使用JsonPath，你可以方便的查找节点、获取想要的数据，JsonPath是Json版的XPath。JSONPath语法参考：[点击这里](https://restfulapi.net/json-jsonpath/) [点击这里](https://www.cnblogs.com/youring2/p/10942728.html)

> XPath 使用路径表达式来选取 XML 文档中的节点或节点集。XPath语法参考：[点击这里](https://www.w3school.com.cn/xpath/index.asp)

### 1.1 插件详细解说

安装插件：

![img](https://img2020.cnblogs.com/blog/1740081/202011/1740081-20201128154151583-370702470.png)

插件具体配置：

![img](https://img2020.cnblogs.com/blog/1740081/202011/1740081-20201128154930398-2041412999.png)

 设置验证时的token值：

![img](https://img2020.cnblogs.com/blog/1740081/202011/1740081-20201128155201223-1232697626.png)

 设定返回状态和设置阈值：

![img](https://img2020.cnblogs.com/blog/1740081/202011/1740081-20201128162712601-451489173.png)

## 2. jira配合Generic Webhook Trigger

### 2.1 配置jira的wenhook

![img](https://img2020.cnblogs.com/blog/1740081/202011/1740081-20201128163458279-440396678.png)

 

### 2.2 创建Jenkins新job

在第一次使用时先不要设置阈值，直接使用token触发工程，然后从Jenkins日志中获取反馈信息。

![img](https://img2020.cnblogs.com/blog/1740081/202011/1740081-20201128164105583-1848548062.png)

 在第一行的信息转换为json格式，就可以看到触发信息，然后从中获取你想要的阈值进行过滤。

## 3. 在pipeline中配置Generic Webhook Trigger

参考地址：[点击这里](https://gitbook.curiouser.top/origin/jenkins-generic-webhook-trigger插件.html) [点击这里](https://www.qedev.com/linux/15881.html)

注意：webhook插件可以在pipeline中直接使用，**只要pipeline工程被执行，pipeline外面的插件配置就会生效。**

- **情况一：配置完成后点击立即构建需要加载第一次会发生错误，第二次立即构建会在插件中自动生成配置。**
- **情况二：配置修改，第一次修改触发pipeline在生成配置工程会报错，第二次构建触发会触发成功。**

**3.1 配置pipeline**

**只需要在pipeline中配置，然后构建生成，插件信息会自动在插件中生成。pipeline中的配置和插件配置相似，只是把改成了文字格式。**

```
声明式语法实例：
pipeline {
  agent any
  triggers {
    GenericTrigger(
     genericVariables: [
      [key: 'ref', value: '$.ref'],
      //[key: 'fixversion', value: '$.issue.fields.customfield_10415']
     ],
     token: 'jenkins',
     printContributedVariables: true,
     printPostContent: true,
     silentResponse: false,
     regexpFilterText: '',
     regexpFilterExpression: ''
    )
  }
  stages {
    stage('test') {
      steps {
        sh "echo $ref"
      }
    }
  }
}
```

### 3.2 生成配置

配置pipeline ---> 点击立即构建 ---> 查看日志(第一行 )

**注：当配置没有生成检查语法是否正确，比如少个逗号。[官方配置](https://plugins.jenkins.io/generic-webhook-trigger/)在最下面有对pipeline有详细解释。**