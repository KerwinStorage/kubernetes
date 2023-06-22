```
sudo docker run -d --name jenkins \
    --restart always \
    -e "JENKINS_OPTS=--httpPort=8089 --httpsPort=-1" \
    -e JAVA_OPTS="-Dhudson.model.ParametersAction.keepUndefinedParameters=true -Duser.timezone=Asia/Shanghai" \
    -e "JENKINS_URL=http://192.168.72.129" \
    -e "BUILD_URL=http://192.168.72.129" \
    -e "JOB_URL_URL=http://192.168.72.129" \
    --restart always \
    --network host \
    -v /var/run/docker.sock:/var/run/docker.sock  \
    -v /usr/bin/docker:/usr/bin/docker  \
    -v jenkins_home:/var/jenkins_home \
    jenkins/jenkins:lts
```

 