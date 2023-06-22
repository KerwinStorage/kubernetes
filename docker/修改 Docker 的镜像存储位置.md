```bash
# 停止 Docker
systemctl stop  docker.service

# 磁盘内选定存放位置/data
# 迁移到/data
mv /var/lib/docker /data

# 建立软连接
sudo ln -s /data/docker /var/lib/docker

# 启动 Docker
systemctl start  docker.service
```

 