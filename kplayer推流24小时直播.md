# kplayer推流24小时直播
## 简介
KPlayer是由ByteLang Studio设计开发的一款用于在Linux环境下进行媒体资源推流的应用程序。

只需要简单的修改配置文件即可达到开箱即用的目的，不需要了解众多推流适配、视频编解码的细节即可方便的将媒体资源在主流直播平台上进行直播。意愿是提供一个简单易上手、扩展丰富、性能优秀适合长时间不间断推流的直播推流场景。
- kplayer官方文档：https://docs.kplayer.net/v0.5.8/overview/
## 步骤
1. 购买或租用一个云服务器，部署linux系统，或使用虚拟机。
阿里云、腾讯云、京东云等等都可以，只要部署Linux系统就可以使用。
2. 使用脚本拉取安装kplayer整合包
- 进入合适的目录，执行一键下载安装脚本
    ```
    curl -fsSL get.kplayer.net | bash
    ```
- 输出结果
    ```
    > curl -fsSL get.kplayer.net | bash
    % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
    100 24.4M  100 24.4M    0     0  7377k      0  0:00:03  0:00:03 --:--:-- 7379k
    kplayer/
    kplayer/kplayer
    kplayer/config.json.example
    ```
3. 获取本人直播推流码
- 以某牙直播为例，获取方式
    - 个人中心-->主播设置-->普通/VR远程推流-->获取推流地址
    ```
    ```
4. 修改上传*配置文件*
- 文件名：config.json
- 关键参数：
    - resource里的 "lists":存放视频的文件夹路径，一般放在kplayer文件夹下的video文件夹，如/root/docker_data/kplayer/video/。
    - output里的"lists"中"path"路径：上边获取到的推流地址，如果获取分为服务器+推流码，直接使用'/'拼接即可。
    - play_model：loop 为循环播放。
    - show-filename：显示正在播放的文件名称插件
    - show-progress：显示正在播放的时间进度插件
- 完整代码
    ```json
    {
    "version": "2.0.0",
    "resource": {
      
      "lists": ["/root/docker_data/kplayer/video/"],
      
      "extensions": []
    },
    "output": {
      "lists": [
        {
          
          "path": "rtmp://hs.direct.huya.com/huyalive/167693323-167693323-7412623593954860791-335510102-10057-A-1725885974-1?seq=1726042321666&type=simple"
        }
      ],
      "reconnect_internal": 5
    },
    "play": {
      "fill_strategy": "ratio",
      "skip_invalid_resource": true,
      
      "cache_on": true,
      
      "play_model": "loop"
    },
    "plugin": {
    "lists": [
      {
        "path": "show-filename",
        "unique": "my_plugin",
        "params": {
          "fontcolor": "red"
        }
      },
      {
        "path": "show-progress",
        "unique": "my_showprogress",
        "params": {
          "fontcolor": "red",
          "y":"30"
        }
      }
    ]
  }
  }
  
  ```
- 把该配置文件上传到服务器kplayer文件夹下。    
5. 上传视频文件到服务器中由配置文件指定的文件夹内  
如：/root/docker_data/kplayer/video/

6. 启动kplayer推流
- 直接启动，若关闭ssh连接界面，推流停止    
    ```
    ./kplayer play start
    ```
- 后台运行，24小时推流
    ```
    ./kplayer play start --daemon
    ```
## 查看直播间
https://www.douyu.com/12231881?dyshid=3f7d06c-20de7bb8f9f6e1823416232200091601&dyshci=208

## 使用docker部署
---
### 步骤
1. 创建kplayer文件夹
  ```
  // 存放视频文件
mkdir -p /root/docker_data/kplayer1/video

// 存放播放时的缓存数据
mkdir -p /root/docker_data/kplayer1/cache
  ```
2. 进入文件夹
  ```
  cd /root/docker_data/kplayer
  ```
3. 编辑config.json和docker-compose.yml并上传，docker-compose.yml如下
  ```
  version: "3.3"
services:
  kplayer:
    container_name: kplayer1
    volumes:
      - "/root/docker_data/kplayer1/video:/video"
      - "/root/docker_data/kplayer1/config.json:/kplayer/config.json"
      - "/root/docker_data/kplayer1/cache:/kplayer/cache"
    restart: always
    image: "bytelang/kplayer:latest"
  ```
4. 运行
>- 运行yml文件
>
>- 进入 /root/docker_data/kplayer 文件夹下面，运行命令：docker-compose up -d
>
>- 或者在任意文件夹下面，运行命令：docker-compose -f /root/docker_data/kplayer/docker-compose.yml up -d
>  docker-compose -f /root/docker_data/kplayer2/docker-compose.yml up -d
>
>  
### 常用docker指令

> docker exec -it kplayer  kplayer resource current   查看当前播放资源
>
> docker exec -it kplayer kplayer plugin update my_plugin --param "fontsize=30"   更新插件参数
> docker exec -it kplayer kplayer plugin list 查看插件列表
>
> cd /root/docker_data/kplayer
>
> docker-compose -f /root/docker_data/kplayer/docker-compose.yml up -d 启动容器
>
> docker logs kplayer 查看日志
>
> docker ps 能查看到启动的容器
>
> docker stop kplayer1  # 停止容器
> docker rm kplayer1    # 删除容器（可选）
>
> 