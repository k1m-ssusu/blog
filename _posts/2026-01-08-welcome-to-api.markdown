---
layout:	post
title:	"Welcome to API!!!"
data:	2026-01-08 22:29:20 +0800
categories: jekyll updata
---
![API 接口使用预览]({{ site.baseurl }}/assets/images/posts/api-20260108.png)

Nginx + Python（后端）+ Maridb + Redis

windows 安装启动 Nginx [Nginx: 下载和部署](https://nginx.org/en/download.html)
windows 安装启动 Apache [Apache-http:下载和部署](https://httpd.apache.org/)

```
# nginx 启动方式 windows 进入 nginx 根目录 注意端口问题
start .\nginx.exe
.\nginx.exe -s reload
.\nginx.exe -t 
netstat -ano | findstr 8080
tasklist | findstr nginx
taskkill /f /im nginx.exe
taskkill /f /pid xxx
Get-Process -PID xxxx 
```

nginx.conf 配置

```
# 反向代理（8080 -> /api/ -> 127.0.0.1:5001）
upstream backend_server {
    server 127.0.0.1:5001;
}
server {
    listen   8080;
    server_name  localhost;
    root  C:\Users\couxi\Desktop\project;  # 正确的路径格式（/分隔符+绝对路径）

    # 反向代理核心配置（删除冗余的root）
    location / {
        proxy_pass http://127.0.0.1:80/;
        # 关键：指定HTTP 1.1协议（避免代理异常）
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        转发客户端真实IP和Host（后端服务需要获取真实IP时必须）
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
         代理超时配置（避免504错误）
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 60s;
   }
    location /api/ {
        proxy_pass http://backend_server;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   F:/nginx-1.28.1/html;  # 改为nginx安装目录下的html绝对路径
    }

}
```

后端接口：Python [Python:下载和部署](https://www.python.org/)

```
C:.
│  app.py
│  db.py
│  zhiyuan_chat.py
│
├─html
│      indx.html
│
└─__pycache__
        db.cpython-313.pyc
        zhiyuan_chat.cpython-313.pyc
```

```
# app后端主要文件
# 下载python
# pip install Flask request jsnify
# pin install pymysql
=======================================================
from flask import Flask,request,jsonify
from zhiyuan_chat import call_chat_zhiyuan_api
from db import save_chat
import pymysql
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

app = Flask(__name__)

# /api/chat endpoint
@app.route("/api/chat", methods=['POST'])
def chat():
# data = request.get_json()
# msg = data.get("message", "")
# return jsonify({"reply": f"You said: {msg}"})

    msg = request.json["message"]

    cached = r.get(msg)
    if cached:
        return jsonify({"reply": cached,"from":"redis"})
    reply = call_chat_zhiyuan_api(msg)
    save_chat(msg, reply)

    r.setex(msg, 300, reply)
    return jsonify({"reply": reply})

# /api/history endpoint
@app.route("/api/history", methods=['GET'])
def get_db():
    return pymysql.connect(
        host='127.0.0.1',
        user="root",
        password='wxl990099.',
        database='chatdb',
        charset='utf8mb4',
        cursorclass=pymysql.cursors.DictCursor
    )
def get_history():
    db = get_db()
    try:
        with db.cursor() as cursor:
            sql = """
            SELECT id,user_msg,reply,created_at
            FROM chat_history
            ORDER BY created_at DESC
            LIMIT 20
            """
            cursor.execute(sql)
            row = cursor.fetchall()
            return jsonify({"code":0,"data":row})
    finally:
        db.close()
if __name__ == "__main__":
  
    app.run(host="127.0.0.1", port=5001,debug=True)
```

```
vimzhiyuan_api.py 后端 api 接口
==============================================
import os 
import requests

ZHIYUAN_API_KEY = os.getenv("ZHIYUAN_API_KEY")
ZHIYUAN_API_URL = "https://open.bigmodel.cn/api/paas/v4/chat/completions"

def call_chat_zhiyuan_api(msg:str) -> str:
    if not ZHIYUAN_API_KEY:
        raise RuntimeWarning("ZHIYUAN_API_KEY is not set")
    headers = { "Authorization" : f"Bearer {ZHIYUAN_API_KEY}", "Content-Type": "application/json" }
    payload = {
        "model": "glm-4.5-flash",
        "messages": [
            {"role": "user", "content": msg}
        ],
        "max_tokens": 2048,
        "temperature": 0.7
    }
    response = requests.post(ZHIYUAN_API_URL, headers=headers,json=payload,timeout=15)
    response.raise_for_status()
    data = response.json()
    return data["choices"][0]["message"]["content"]
```

```
# vim db.py 后端数据库接口
=================================================
import pymysql
conn = pymysql.connect(
    host='localhost',
    user='root',
    password='wxl990099.',
    database='chatdb',
    charset='utf8mb4'
)
def save_chat(use_msg, reply):
    with conn.cursor() as cursor:
        sql = "INSERT INTO chat_history (use_msg, reply) VALUES (%s, %s)"
        cursor.execute(sql, (use_msg, reply))
        conn.commit()
```

```
vim index.html 前端接口==需要与 nginx 的配置一致
nginx : root  F:/www/html
apache : DocumentRoot "F:\www\html" <Directory "F:\www\html"></Directory>
=========================================================
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>chat_test</title>
</head>
<body>
    <h2>chat_demo</h2>

    <input id="msg" type="text" placeholder="enter your message here" style="width:300px;">
    <button onclick="send()">send</button>
    <pre id="output"></pre>
    <script>
        function send() {
            const msg = document.getElementById("msg").value;
            fetch("/api/chat", {
                method: "POST",
                headers: {
                    "Content-Type": "application/json"
                },
                body: JSON.stringify({message:msg})
            }) .then(res => res.json())
               .then(data => {
                   document.getElementById("output").textContent += "User: " + msg + "\n" + "Bot: " + data.reply + "\n\n";
                });
            }
        function loadHistory() {
            fetch("/api/history")
                .then(res => res.json())
                .then(data => {
                    const url = document.getElementById("history");
                    ul.innerHTML = "";
                    data.data.forEach(item => {
                        const li = document.createElement("li");
                        li.innerText = `[${item.created_at}] User: ${item.user_msg} | Bot: ${item.reply}`;
                        ul.appendChild(li);
                    });
                });
            }   
</script>
</body>
</html>

```

Mariadb  [数据库Mariadb下载部署](https://mariadb.com/docs/server/clients-and-utilities/legacy-clients-and-utilities/mysql_install_db)

```
mysql -u root -p
# 创建项目所需要的数据库
# 数据存储
========================
CREATE DATABASE chatdb DEFAULT CHARSET utf8mb4;
USE chatdb;
CREATE TABLE chat_history (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_msg TEXT NOT NULL,
    reply TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

```

Redis 缓存机制（内存缓存）[Redis 官网 ](https://typing.io/lesson/c/redis/db.c/1)

```
# 启动redis 缓存
.\redis-server.exe
[17252] 09 Jan 01:03:55.789 # Warning: no config file specified, using the default config. In order to specify a config file use C:\Program Files\Redis\redis-server.exe /path/to/redis.conf
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 3.0.504 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 17252
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[17252] 09 Jan 01:03:55.793 # Server started, Redis version 3.0.504
[17252] 09 Jan 01:03:55.793 * DB loaded from disk: 0.000 seconds
[17252] 09 Jan 01:03:55.793 * The server is now ready to accept connections on port 6379
```

部署过程中存在问题

```
一、httpd.conf 和 nginx.conf的 html 的位置存放 windos
二、python-Flask 的 5000 端口经常出现僵尸进程，无法再次利用端口只能强制关闭重启，可以换成 5001
三、nginx.conf 的配置如何能够反向代理 /api/ 到指定 5000 的端口
四、windows 发起请求的命令 (Invoke-WebRequest -Uri "http://127.0.0.1:8080/api/chat" -Method POST -Headers @{"Content-Type"="application/json"} -Body '{"message": "你好，介绍一下自己"}').Content
五、API 密钥存放在环境中 set ZHIYUAN_API_KEY="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" 
六、本次使用的智源 API 进行文本对话
```
