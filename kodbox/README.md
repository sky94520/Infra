# 1. 简介
目前市面上已经有成熟的私有云盘，比如可道云、cloudreve等。在选择并试用了几个产品后，最终选择可道云。选择可道云的理由有以下几个：
1. 支持MinIO - 因为我已经搭建了MinIO，所以比较偏向于使用MinIO作为存储；
2. 多端支持 - Android、IOS、Web；
3. 花里胡哨的界面。
# 2. 部署
可道云有提供docker镜像，直接使用该镜像，并作为Deployment就可以完成简单地部署，但是存在的问题就是配置丢失，比如配置了MySQL、Redis，以及升级，插件等。
为了避免容器重启造成配置丢失的问题，我这里选择使用StatefulSet + PVC，使用PVC保存配置，避免容器重启后配置消失的问题。
总得来说，需要创建一个StatefulSet运行可道云程序，并搭配PVC保存配置，同时使用一个Service对外提供服务。
首先分析下docker镜像[kodbox-image](https://hub.docker.com/layers/kodcloud/kodbox/v1.4505/images/sha256-16267afc9cf0c6c988824c5aa81702210e5bfada49908ba8580c63fff09efa12?context=explore)，在Imager Layer Details的最后，有这样两条命令：
```docker
ENTRYPOINT ["/entrypoint.sh"]
CMD ["/usr/bin/supervisord" "-n" "-c" "/etc/supervisord.conf"]
```
如果对docker镜像有了解的话，直接执行这个镜像得行为是先执行脚本```/entrypoint.sh```，然后再执行CMD。entrypoint.sh的文件内容如下：
```shell
#!/bin/sh
set -eu

# return true if specified directory is empty
directory_empty() {
  [ -z "$(ls -A "$1/")" ]
}

waiting_for_connection(){
  until nc -z -w 3 "$1" "$2"; do
    >&2 echo "Waiting for connection to the $1 host on port $2"
    sleep 1
  done
}

file_env() {
  local var="$1"
  local fileVar="${var}_FILE"
  local def="${2:-}"
  local varValue=$(env | grep -E "^${var}=" | sed -E -e "s/^${var}=//")
  local fileVarValue=$(env | grep -E "^${fileVar}=" | sed -E -e "s/^${fileVar}=//")
  if [ -n "${varValue}" ] && [ -n "${fileVarValue}" ]; then
    echo >&2 "error: both $var and $fileVar are set (but are exclusive)"
    exit 1
  fi
  if [ -n "${varValue}" ]; then
    export "$var"="${varValue}"
  elif [ -n "${fileVarValue}" ]; then
    export "$var"="$(cat "${fileVarValue}")"
  elif [ -n "${def}" ]; then
    export "$var"="$def"
  fi
  unset "$fileVar"
}

file_env MYSQL_SERVER
file_env MYSQL_DATABASE
file_env MYSQL_USER
file_env MYSQL_PASSWORD
file_env MYSQL_PORT
file_env CACHE_TYPE
file_env CACHE_HOST
file_env CACHE_PORT
file_env KODBOX_ADMIN_USER
file_env KODBOX_ADMIN_PASSWORD
file_env FPM_MAX
file_env FPM_START
file_env FPM_MIN_SPARE
file_env FPM_MAX_SPARE

MYSQL_PORT=${MYSQL_PORT:-3306}
CACHE_TYPE=${CACHE_TYPE:-redis}
CACHE_PORT=${CACHE_PORT:-6379}

FPM_MAX=${FPM_MAX:-50}
FPM_START=${FPM_START:-10}
FPM_MIN_SPARE=${FPM_MIN_SPARE:-10}
FPM_MAX_SPARE=${FPM_MAX_SPARE:-30}

waiting_for_db(){
  waiting_for_connection $MYSQL_SERVER $MYSQL_PORT
}

waiting_for_cache(){
  waiting_for_connection $CACHE_HOST $CACHE_PORT
}

CONIG_FILE=/usr/src/kodbox/config/setting_user.php

if [ -n "${MYSQL_DATABASE+x}" ] && [ -n "${MYSQL_USER+x}" ] && [ -n "${MYSQL_PASSWORD+x}" ] && [ ! -f "$CONIG_FILE" ]; then
  mv /usr/src/kodbox/config/setting_user.example $CONIG_FILE
  sed -i "s/MYSQL_SERVER/${MYSQL_SERVER}/g" $CONIG_FILE
  sed -i "s/MYSQL_DATABASE/${MYSQL_DATABASE}/g" $CONIG_FILE
  sed -i "s/MYSQL_USER/${MYSQL_USER}/g" $CONIG_FILE
  sed -i "N;6 a 'DB_PWD' => '${MYSQL_PASSWORD}'," $CONIG_FILE
  sed -i "s/MYSQL_PORT/${MYSQL_PORT}/g" $CONIG_FILE
  touch /usr/src/kodbox/data/system/fastinstall.lock
  if [ -n "${KODBOX_ADMIN_USER+x}" ] && [ -n "${KODBOX_ADMIN_PASSWORD+x}" ]; then
    echo -e "ADM_NAME=${KODBOX_ADMIN_USER}\nADM_PWD=${KODBOX_ADMIN_PASSWORD}" >> /usr/src/kodbox/data/system/fastinstall.lock
  fi
  if [ -n "${CACHE_HOST+x}" ]; then
    sed -i "s/CACHE_TYPE/${CACHE_TYPE}/g" $CONIG_FILE
    sed -i "s/CACHE_HOST/${CACHE_HOST}/g" $CONIG_FILE
    sed -i "s/CACHE_PORT/${CACHE_PORT}/g" $CONIG_FILE
  else
    sed -i "s/CACHE_TYPE/file/g" $CONIG_FILE
    sed -i "s/CACHE_HOST/file/g" $CONIG_FILE
  fi
fi

if [ -n "${PUID+x}" ]; then
  if [ ! -n "${PGID+x}" ]; then
    PGID=${PUID}
  fi
  deluser nginx
  addgroup -g ${PGID} nginx
  adduser -D -S -h /var/cache/nginx -s /sbin/nologin -G nginx -u ${PUID} nginx
  chown -R nginx:nginx /var/lib/nginx/
fi

if [ -n "${FPM_MAX+x}" ]; then
  sed -i "s/pm.max_children = .*/pm.max_children = ${FPM_MAX}/g" /usr/local/etc/php-fpm.d/www.conf
  sed -i "s/pm.start_servers = .*/pm.start_servers = ${FPM_START}/g" /usr/local/etc/php-fpm.d/www.conf
  sed -i "s/pm.min_spare_servers = .*/pm.min_spare_servers = ${FPM_MIN_SPARE}/g" /usr/local/etc/php-fpm.d/www.conf
  sed -i "s/pm.max_spare_servers = .*/pm.max_spare_servers = ${FPM_MAX_SPARE}/g" /usr/local/etc/php-fpm.d/www.conf
fi

if  directory_empty "/var/www/html"; then
  if [ "$(id -u)" = 0 ]; then
    rsync_options="-rlDog --chown nginx:nginx"
  else
    rsync_options="-rlD"
  fi
  echo "KODBOX is installing ..."
  rsync $rsync_options --delete /usr/src/kodbox/ /var/www/html/
  if [ -f "$CONIG_FILE" ]; then
    if [ -n "${CACHE_HOST+x}" ]; then
      waiting_for_cache
    fi
    waiting_for_db
    php /var/www/html/index.php "install/index/auto"
    chown -R nginx:root /var/www
  fi
else
  echo "KODBOX has been configured!"
fi

if [ -f /etc/nginx/ssl/fullchain.pem ] && [ -f /etc/nginx/ssl/privkey.pem ] && [ ! -f /etc/nginx/sites-enabled/*-ssl.conf ] ; then
  ln -s /etc/nginx/sites-available/private-ssl.conf /etc/nginx/sites-enabled/
  sed -i "s/#return 301/return 301/g" /etc/nginx/nginx.conf
fi

exec "$@"
```
```entrypoint.sh```这个脚本用于安装KODBOX，包括配置数据库、缓存等。首先会检查是否需要更新配置文件setting_user.php并修改其中的参数。然后根据环境变量来设定nginx和php-fpm的用户权限以及进程管理器相关参数，最后执行KODBOX安装过程。
如果想要实现缓存配置，就不能每次容器重启的时候都执行这个脚本。因此，我们在编写StatefulSet的时候，需要判断一下```/var/www/html```文件夹是否存在来判断是否是第一次启动，只有第一次启动得时候才会执行```entrypoint.sh```脚本，否则直接启动可道云。
Statefulset的内容如下：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kodbox-svc
  namespace: default
spec:
  clusterIP: None
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
  selector:
    app: kodbox-svc
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kodbox
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kodbox-svc
  template:
    metadata:
      labels:
        app: kodbox-svc
    spec:
      initContainers:
      - name: volume-mount-hack
        image: busybox
        command: ["sh", "-c", "chmod -f -R 777 /var/www/html"]
        volumeMounts:
        - name: kodbox-data
          mountPath: /var/www/html
      containers:
        - name: kodbox
          image: kodcloud/kodbox:v1.4505
          command:
          - "/bin/sh"
          - "-c"
          - |
            if [ -d "/var/www/html/config" ]; then
              echo "config exist, use old config"
            else
              /entrypoint.sh
            fi
            /usr/bin/supervisord -n -c /etc/supervisord.conf
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          securityContext:
            privileged: true
          volumeMounts:
            - name: kodbox-data
              mountPath: /var/www/html
  volumeClaimTemplates:
  - metadata:
      name: kodbox-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: nfs
      resources:
        requests:
          storage: 1Gi
```
StatefulSet有两个containers：
1. volume-mount-hack - ```initContainers```的作用是设置对```/var/www/html```的访问权限，避免kodbox在执行的时候，存在权限不足的问题；
2. kodbox - 它首先会判断是否已经存在了```/var/www/html```文件夹，不存在才执行```/entrypoint.sh```，在保证配置文件都存在的情况下，在执行可道云的启动命令。
:::tip
上面申请挂载的Volume来自于storageClass:nfs，nfs是由nfs服务器生成的StorageClass。在我搭建可道云的使用，使用nfs确实会报权限的问题，不确定其他的PV是否存在类似的问题。
:::
为了向外界提供服务，还需要写一个Service：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kodbox-nodeport-svc
  namespace: default
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
  selector:
    app: kodbox
```
执行完上述的文件后，可以查看k8s给这个service分配的端口，然后在局域网内就可以访问可道云啦，在配置好MySQL、Redis后，可以尝试主动删除这个pod，来确定配置没有丢失。
# 3. 公开服务
公开可道云，我尝试使用cpolar和zerotier，下面直接说结论：
- cpolar是传统的内网穿透服务，它的优点是有一个较为稳定的域名，缺点是下载上行带宽会受到影响。如果想要公开服务，推荐使用内网穿透方案。
- zerotier 据说是P2P，免费使用，但是只能是配置了同一虚拟网络下的终端才可以访问，网络不太稳定，据说有个公有IP的机器搭建Moon可以提升速度和稳定性，但是我没有公网IP的服务器。如果只是确定人数的访问，推荐zerotier。

**zerotier参考链接：**
1. [Zerotier 安装使用保姆级心得2023(上篇）](https://post.smzdm.com/p/awkom242/)
2. [ZeroTier安装与配置](https://makefile.so/2022/05/27/install-zerotier/)
# 4. 参考链接
1. [对比9大开源网盘程序，自建网盘指南](https://www.bilibili.com/video/BV1mF41137Vc/?spm_id_from=333.337.search-card.all.click&vd_source=46bdf472acaa4208abb11baa3514949e)
2. [白嫖可道云静态文件CDN，给自建站点加速](https://www.9kr.cc/archives/54/)
3. [Docker安装的KodBox更新插件时报错，请问如何解决？](https://bbs.kodcloud.com/d/3144-dockeran-zhuang-de-kodboxgeng-xin-cha-jian-shi-bao-cuo-qing-wen-ru-he-jie-jue)
4. [Kubernetes Permission denied for mounted nfs volume](https://stackoverflow.com/questions/50854701/kubernetes-permission-denied-for-mounted-nfs-volume)