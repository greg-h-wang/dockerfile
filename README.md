Dockerfiles: 标准镜像、基础镜像和服务镜像
===

这是一个包含所有公用镜像构建过程的仓库，主要分三部分：Dockerfile、公共资源、构建脚本。

所有维护这个仓库的人员都应该提供并及时更新自己的文档。

![Docker logo](resources/static_files/docker-logo-compressed.png "Docker")

## Dockerfile
---

包含：library、base、service

结构：

```bash
.
|-- base # 父类
|   |-- centos # 子类
|   |   `-- centos=version-zh.df # 模糊版本
|   |-- gradle
|   |   `-- gradle=2.14.1.df # 明确版本
|   |-- jdk
|   |   `-- jdk=8u101.df
|   `-- tomcat
|       |-- src # 私有资源（可选）
|       |   |-- docker-entrypoint-kazoo.sh
|       |   `-- docker-entrypoint.sh
|       |-- tomcat=8.0.36.df
|-- library
|   |-- centos
|       |-- centos=7.2.1511.df
|       |-- centos=latest.df # 最后版本
|       `-- src
|           `-- centos-7.2.1511-docker.tar.xz
|-- service
    |-- elasticsearch
        |-- elasticsearch=2.3.5.df
        `-- src
            `-- docker-entrypoint.sh

- library：包含各类标准镜像，基本上属于官方的标准流程，该动极少且不做定制。

- base：包含各类基础镜像，根据需求做定制，不修改原有结构。

- service：包含各类服务镜像，根据需求做定制，尽量不修改原有结构。
```

用途：规整镜像Dockerfile的存放结构

## 公共资源
---

包含：resources

结构：

```bash
.
`-- sources
    |-- apache-tomcat-8.0.36.tar.gz
    |-- elasticsearch-2.3.5.tar.gz
    |-- gradle-2.14.1-bin.zip
    `-- jdk-8u101-linux-x64.tar.gz
```

用途：存放镜像构建过程中的依赖文件

## 构建脚本
---

包含：build.sh、build.list

结构：

- build.sh:

```bash
#!/bin/bash

cd `dirname ${BASH_SOURCE}` # 切换到工作目录

suffix="2257b4561a43e400e437b662529255e3" # 标签

[ -n "${1}" ] && DOCKER_REGISTRY="${1%%/*}/" # docker仓库
os="centos"
fs="simplehttpserver"
dns="bind"
mnt=`pwd`/resources
DOCKER_BUILD_LIST=`grep -v '^ *#' build.list` # 取得任务

# 构建最简单的系统镜像
sed -i '/^FROM/d' library/${os}/${os}=latest.df
sed -i "1iFROM scratch" library/${os}/${os}=latest.df
docker build -t ${os}:latest -f library/${os}/${os}=latest.df library/${os}

# 构建最简单的文件服务镜像
sed -i '/^FROM/d' library/${fs}/${fs}=latest.df
sed -i "1iFROM ${os}" library/${fs}/${fs}=latest.df
docker build -t ${fs}:latest -f library/${fs}/${fs}=latest.df library/${fs}

# 构建最简单的解析服务镜像
sed -i '/^FROM/d' library/${dns}/${dns}=latest.df
sed -i "1iFROM ${os}" library/${dns}/${dns}=latest.df
docker build -t ${dns}:latest -f library/${dns}/${dns}=latest.df library/${dns}

# 运行文件服务并挂载资源
docker stop ${fs}-${suffix} && docker rm ${fs}-${suffix} || echo
docker run -d -P --name ${fs}-${suffix} -v ${mnt}:/mnt ${fs}:latest
fsIp=`docker inspect -f '{{.NetworkSettings.IPAddress}}' ${fs}-${suffix}`

# 运行解析服务
docker stop ${dns}-${suffix} && docker rm ${dns}-${suffix} || echo
docker run -d -P --name ${dns}-${suffix} ${dns}:latest
docker exec ${dns}-${suffix} bash -c "echo '${fs} A ${fsIp}' >> /var/named/named.localhost"
docker restart ${dns}-${suffix}
dnsIp=`docker inspect -f '{{.NetworkSettings.IPAddress}}' ${dns}-${suffix}`

# 修改宿主机dns
sed -i "/${suffix}/d" /etc/resolv.conf
sed -i "1inameserver ${dnsIp} # ${suffix}" /etc/resolv.conf

# 构建镜像
for LINE in ${DOCKER_BUILD_LIST}
do
    FROM_IMAGE=`echo ${LINE} | awk -F',' '{print $1}'`
    TO_IMAGE=`echo ${LINE} | awk -F',' '{print $2}'`
    DOCKERFILE_FILE=`echo ${LINE} | awk -F',' '{print $3}'`
    DOCKERFILE_PATH=${DOCKERFILE_FILE%/*}

    [ -n "${DOCKER_REGISTRY}" ] && docker pull ${DOCKER_REGISTRY}${TO_IMAGE} || echo "${DOCKER_REGISTRY}${TO_IMAGE} not found!"
    sed -i '/^FROM/d' ${DOCKERFILE_FILE}
    if [ 'scratch' != "${FROM_IMAGE}" ]
    then
        sed -i "1iFROM ${DOCKER_REGISTRY}${FROM_IMAGE}" ${DOCKERFILE_FILE}
    else
        sed -i "1iFROM ${FROM_IMAGE}" ${DOCKERFILE_FILE}
    fi
    docker build -t ${DOCKER_REGISTRY}${TO_IMAGE} -f ${DOCKERFILE_FILE} ${DOCKERFILE_PATH}
    [ -n "${DOCKER_REGISTRY}" ] && docker push ${DOCKER_REGISTRY}${TO_IMAGE}
done

# 清理
sed -i "/${suffix}/d" /etc/resolv.conf
docker stop ${dns}-${suffix} && docker rm ${dns}-${suffix} || echo
docker stop ${fs}-${suffix} && docker rm ${fs}-${suffix} || echo
```

- build.list:
```bash
scratch,library/centos:7.2.1511,library/centos/centos=7.2.1511.df
base/centos:7.2.1511-zh,base/jdk:8u101,base/jdk/jdk=8u101.df
base/jdk:8u101,base/gradle:2.14.1-8u101,base/gradle/gradle=2.14.1.df
base/jdk:8u101,base/tomcat:8.0.36-8u101,base/tomcat/tomcat=8.0.36.df

- 以","分隔为三部分。第一部分：上层镜像，第二部分：目标镜像，第三部分：Dockerfile位置。
```

用途：执行build.sh，根据build.list文件所指定的关系，依次构建镜像。
