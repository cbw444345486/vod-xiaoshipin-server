


# 概述

本项目为腾讯云小视频 APP 后台服务，采用 Nodejs 和 MySQL 搭建，提供了如下功能的演示：
1. 上传签名派发
2. 媒资管理
3. 内容审核

用户可以下载本项目源码，快速搭建自己的小视频后台服务。


# 准备

## 帐号申请
申请[腾讯云](https://cloud.tencent.com/)帐号，获取[ API 密钥](https://console.cloud.tencent.com/cam/capi)，得到 Appid、SecretId、SecretKey。

## 环境准备

### 安装 Nodejs
注意：nodejs 版本要求高于 8.x。
```
curl -sL https://deb.nodesource.com/setup_9.x | sudo -E bash -
sudo apt-get install -y nodejs
```



### 安装 MySQL (mariadb) 
```
sudo apt update
sudo apt install mariadb-server
sudo mysql --version
sudo service mysql start
```
### 初始化数据库

1. 在终端使用 root 帐号登录 MySQL：
```
sudo mysql -u root -p
```
2. 创建小视频数据库用户 litvideo：

```
create user 'litvideo'@'localhost' identified by 'litvideo';
```

3. 创建小视频数据库 db_litvideo，并授权给小视频用户 litvideo：

```
create database db_litvideo default charset utf8 collate utf8_general_ci;
grant all privileges on `db_litvideo`.* to 'litvideo'@'%' identified by 'litvideo';
```

4. 使用 litvideo，新建所需要的数据库：
```
use db_litvideo;
CREATE TABLE IF NOT EXISTS tb_account(
  userid VARCHAR(50) NOT NULL,
  password VARCHAR(255),
  nickname VARCHAR(100),
  sex INT DEFAULT -1,
  avatar VARCHAR(254),
  frontcover varchar(255) DEFAULT NULL,
  create_time timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY(userid)
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS tb_ugc (
  userid varchar(50) NOT NULL,
  file_id varchar(150) NOT NULL,
  title varchar(128) DEFAULT NULL,
  status tinyint(4) not NULL DEFAULT 0,
  review_status tinyint(4) not NULL DEFAULT 0,
  frontcover varchar(255) DEFAULT NULL,
  location varchar(128) DEFAULT NULL,
  play_url varchar(255) DEFAULT NULL,
  create_time timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (file_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS tb_token(
  token VARCHAR(32) NOT NULL,
  userid VARCHAR(50) NOT NULL,
  expire_time DATETIME NOT NULL DEFAULT '1970-01-01',
  refresh_token VARCHAR(32) NOT NULL,
  PRIMARY KEY(token),
  KEY(userid),
  KEY(expire_time)
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS tb_queue(
  task_id VARCHAR(150) NOT NULL,
  file_id VARCHAR(150) NOT NULL,
  owner   VARCHAR(50),
  create_time timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  mark_time timestamp DEFAULT '1971-01-01 00:00:00',
  review_data longtext,
  PRIMARY KEY(task_id)
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS tb_review_record(
  task_id VARCHAR(150) NOT NULL,
  file_id VARCHAR(150) NOT NULL,
  reviewer_id   VARCHAR(50) NOT NULL,
  review_time timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  review_status VARCHAR(50) NOT NULL,
  PRIMARY KEY(task_id)
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```


# 快速开始

## 配置工程
进入工作目录，克隆项目文件：

```
 git clone https://github.com/tencentyun/vod-xiaoshipin-server.git
 cd vod-xiaoshipin-server
```
   
在工作目录下，安装项目所需依赖：
```
npm install            
```
在 conf 文件夹下，复制 config_template.json 文件并命名为 localconfig.json 文件，修改腾讯云 API 密钥、数据库参数配置，以及 COS 存储配置。

```
{
        "dbconfig":{                        //数据库配置
            "host":"127.0.0.1",             //数据库 IP 地址，保持默认本机
            "user":"litvideo",              //数据库用户名，保持默认
            "password":"litvideo",          //数据库登录密码，保持默认
            "database":"db_litvideo",       //小视频所用数据库，保持默认
            "port":3306,                    //数据库端口，保持默认
            "supportBigNumbers": true,      //保持默认
            "connectionLimit":10            //保持默认
        },
        "tencentyunaccount":{                //腾讯运云帐号配置
            "AppId":"",                      //腾讯云 Appid
            "SubAppId":"",                   //腾讯云点播子帐号，默认不使用，保持为空
            "SecretId": "",                  //腾讯云 SecretId
            "SecretKey": "",                 //腾讯云 SecretKey
            "bucket":"xiaoshipin",           // COS 存储bucket
            "region":"ap-guangzhou"          // COS 存储地域
        },
        "server":{                        
            "ip":"0.0.0.0",                 //服务启动 IP ，保持默认
            "port":8001,                    //服务启动端口，保持默认
            "reliablecb":true,               //回调选择，保持默认
            "reliablecbtimeout":5000        //消息拉取轮询间隔（毫秒）
        }
}
```

## 配置可靠回调
在腾讯云点播控制台，【视频处理设置】下【回调配置】中设置回调模式为【可靠回调】，【事件回调配置】中选择【上传完成回调】。详情参考[腾讯云点播回调配置](https://cloud.tencent.com/document/product/266/7829)。
![回调设置](https://main.qcloudimg.com/raw/3dcabb94e5ce7a84c0497cd4c0cb9941.png)


## 启动服务


在工程根目录下启动服务：
```
npm start
```

服务启动后，在另外一个终端下测试视频拉取接口：

```
curl -l -H "Content-type: application/json" -X POST -d '' http://localhost:8001/get_ugc_list
```

如果服务正常运行，可返回如下结果：
```
{"code":200,"message":"OK","data":{"list":[],"total":0}}
```

## 客户端搭建

参考[拥有自己的短视频app-替换代码中的后台地址](https://cloud.tencent.com/document/product/584/15540#step4.-.E6.9B.BF.E6.8D.A2.E7.BB.88.E7.AB.AF.E6.BA.90.E4.BB.A3.E7.A0.81.E4.B8.AD.E7.9A.84.E5.90.8E.E5.8F.B0.E5.9C.B0.E5.9D.80)。


## 功能体验

### 视频上传

服务启动正常后，打开短视频 App，点击底部加号，可以选择“录制”，“视频编辑”和“图片编辑”方式上传视频。上传成功后会在视频列表显示:

![App](https://main.qcloudimg.com/raw/bde628f5f0c9c463e56ffb15710b32ff.png)

用户可上传本项目提供的测试视频，用于体验后续人工审核功能：将项目文件夹 source 中的 video.mp4 导入手机，打开 App，通过“视频编辑”方式上传并发布。

### 内容审核

腾讯云会针对用户上传的视频进行内容审核，审核结果为“pass”的视频可直接在 App 播放，审核结果为 “review”（建议人审）或者 “block”（建议屏蔽）的视频会推到鉴黄墙进行人工审核，打开浏览器访问 http://yourip:port/index.html 即可体验视频审核功能。其中 yourip 为运行本服务的服务器 ip 地址, port 由配置文件 localconfig.json 中，server 的 port 字段定义。页面如图：

![鉴黄墙](https://main.qcloudimg.com/raw/fed1aa095fc4c0d9ba563abd2055775d.png)


页面左侧显示视频 id 和 title，以及触犯规则的视频截图，截图 confidence 超过 70 会标红，右侧支持视频播放。点击相应截图,视频会从指定位置开始播放。 

视频下方是审核通过/屏蔽按钮，审核人点击任一审核按钮后，可获取下一条待审视频。如果没有更多审核任务了，审核完当前视频后会提示：没有更多任务。

### 媒资管理

打开 App，下拉列表会将媒资信息更新到视频列表。

用户上传视频后，会自动发起内容审核，后台根据返回的结果更新媒资，并在 App 列表页面-视频封面图的左上角展示：审核结果为“pass”的显示为“已通过”，“review”和“block”的视频会进行人工审核。

人工审核提交结果后，后台根据审核结果更新媒资，结果为“屏蔽”的视频显示为“涉黄”，“通过”的视频显示为“已审核”。其中只有“已通过”是视频可以播放，其他状态不可播放。如图所示：

![媒资](https://main.qcloudimg.com/raw/e9cdf6f358a075b837a5ea8a51aad563.png)



# 参考
1. 腾讯云点播平台视频上传签名：https://cloud.tencent.com/document/product/266/9219

2. 腾讯云点播平台事件回调：https://cloud.tencent.com/document/product/266/7829

3. 腾讯云点播平台媒资获取：https://cloud.tencent.com/document/product/266/8586

4. 腾讯云点播平台视频审核：https://cloud.tencent.com/document/product/266/17914

5. 腾讯云 Node.js SDK：https://cloud.tencent.com/document/sdk/Node.js
