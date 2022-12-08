#!/usr/bin/env bash

# 1. postgres container
```bash
mkdir -p /opt/sola/postgresql
docker run --name postgres -e POSTGRES_USER=root -e POSTGRES_PASSWORD=abcd.1234 -p 5432:5432 -v /opt/sola/postgresql:/var/lib/postgresql/data -d postgres

#进入容器,创建必要的8个库
/usr/bin/createdb -U root -O root dendrite_userapi
/usr/bin/createdb -U root -O root dendrite_mediaapi
/usr/bin/createdb -U root -O root dendrite_syncapi
/usr/bin/createdb -U root -O root dendrite_room
/usr/bin/createdb -U root -O root dendrite_keyserver
/usr/bin/createdb -U root -O root dendrite_federationapi
/usr/bin/createdb -U root -O root dendrite_appservice
/usr/bin/createdb -U root -O root dendrite_mscs

#/usr/bin/createdb -U root -O root dendrite
```
# 2. pgAdmin4
```bash
docker run -p 5050:80 -e "PGADMIN_DEFAULT_EMAIL=root@sola.com" -e "PGADMIN_DEFAULT_PASSWORD=abcd.1234" -d --name pgadmin4 dpage/pgadmin4

# browser: http://localhost:5050  login ->   root@sola.com abcd.1234
# create connect
#hostName: host.docker.internal
#port: 5432
#Maintenance database name: postgres
#userName: dendrite
#password: itsasecret
```

# 3. create signing key
```bash
cd dendrite
#export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890      #use proxy
./build.sh
./bin/generate-keys --private-key matrix_key.pem
cp ./matrix_key.pem /opt/sola
```

# 4. copy dendrite.yaml and create cert

```bash
注意: alan.tranchegroup.com 443 --> ovpn server --> ovpn client 8088(nginx端口)
global.server_name: alan.tranchegroup.com
global.jetstream.storage_path: /opt/sola/matrix-nats-store
#federation_api.disable_tls_validation: true
#jetstream.disable_tls_validation: true
#app_service_api.disable_tls_validation: true

#mkdir -p /opt/sola/jetstream
#cp /Users/alan/Desktop/source_read/my_github_code/dendrite/dendrite-sample.monolith.yaml /opt/sola/dendrite.yaml
```

# 5. go get 获取项目依赖
go env -w  GOPROXY=https://goproxy.cn,direct
go get

# 6. 启动服务(建议: idea 6GB内存)
```sh
dendrite/cmd/dendrite-monolith-server/main.go

#创建账号
./bin/create-account -url "https://alan.tranchegroup.com" -password "12345678" --config cmd/dendrite-monolith-server/dendrite.yaml --username alan1
./bin/create-account -url "https://alan.tranchegroup.com" -password "12345678" --config cmd/dendrite-monolith-server/dendrite.yaml --username alan2
```

# 7. 部署nginx,做反向代理和配置wellknow端点
```shell
vim /Users/alan/Desktop/source_read/my_github_code/dendrite-10.7/dendrite/cmd/dendrite-monolith-server/conf_dev/conf.d/matrix-domain.conf   #将三处 192.168.1.120 修改为自己局域网ip,方便移动端请求.

docker run --name sola-nginx -p 8088:8088 -v /Users/alan/Desktop/source_read/my_github_code/dendrite-10.7/dendrite/cmd/dendrite-monolith-server/conf_dev/conf.d/matrix-domain.conf:/etc/nginx/conf.d/matrix-domain.conf -d nginx
```

# 网络测试
```shell
#nginx 容器内
curl http://host.docker.internal:8088/.well-known/matrix/client
curl http://host.docker.internal:8088/_matrix/client/versions
#宿主机
curl http://192.168.1.120:8088/.well-known/matrix/client
curl http://192.168.1.120:8088/_matrix/client/versions

#域名
#curl http://10.8.0.2:8088/_matrix/client/versions

curl https://alan.tranchegroup.com/_matrix/client/versions -k
curl https://alan.tranchegroup.com/_matrix/client/versions -k
curl https://alan.tranchegroup.com/.well-known/matrix/client -k
#fluffyChat APP 连接
alan.tranchegroup.com
```
### 操作
```shell
#服务器地址
alan.tranchegroup.com
#邀请创建新的聊天室
@alan1:alan.tranchegroup.com
@alan2:alan.tranchegroup.com
```

# 调优
a. 每个服务数据库连接数
b. nats 集群模式优化
c. 断开matrix联盟


