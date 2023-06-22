## 前言

将sonarqube集成到ldap服务，这样我们可以使用ldap里面的账号进行登录

LDAP插件官方手册文档：https://docs.sonarqube.org/display/SONARQUBE67/LDAP+Plugin

## 主配置文件ldap内容

根据自己的树状图进行修改

```
sonar.security.realm=LDAP
ldap.url=ldap://192.168.10.10:389
ldap.bindDn=cn=admin,dc=dreame,dc=tech
ldap.bindPassword=dreame@2019
ldap.authentication=simple
ldap.user.baseDn=dc=dreame,dc=tech
ldap.user.request=(&(objectClass=inetOrgPerson)(uid={login}))
ldap.user.realNameAttribute=cn
ldap.user.emailAttribute=mail
ldap.group.baseDn=dc=dreame,dc=tech
ldap.group.request=(&(objectClass=groupOfUniqueNames)(uniqueMember={dn}))
```

 