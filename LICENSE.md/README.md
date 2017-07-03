# ss-kcp
* Ubuntu 14.04 64位环境部署**多端口** shadowsocks 服务端

* 启用相对更节约资源的 **CHACHA20** 加密

* 部署更快速的 **kcptun** 服务端

* 在 Android 和 Windows 环境下 **kcptun 客户端** 的配置

------

1. 安装必要组件

    * 更新系统，安装可以直接从 apt 更新的软件：
    
      ```bash
      apt-get update
      apt-get upgrade
      apt-get install build-essential python-pip m2crypto supervisor
      ```
      
   * 安装 shadowsocks：
   
      ```bash
      pip install shadowsocks
      ```
      
   * 安装加密用软件 libsodium：
   
      ```bash
      wget https://github.com/jedisct1/libsodium/releases/download/1.0.11/libsodium-1.0.11.tar.gz
      tar zxvf libsodium-1.0.11.tar.gz
      cd libsodium-1.0.11
      ./configure
      make && make check
      make install
      echo /usr/local/lib > /etc/ld.so.conf.d/usr_local_lib.conf
      ldconfig
      ```

2. 部署 shadowsocks 服务端

   * 编辑 shadowsocks 配置文件：
   
      ```bash
      vi /etc/shadowsocks.json
      ```
      
      *按`I`进入插入模式，粘贴后按`Esc`退出，*
      
      *光标选中需要修改的位置按`X`删除，再进入插入模式修改。*
      
      *最后按`Esc`退出，再输入`:wq`保存退出。*
      
   * 下面配置文件中1080和8080为服务端口号，后面为对应的密码，可以不同。
   
      ```JSON
      {
      "server":"0.0.0.0",
      "port_password":
      {
      "1080":"password1",
      "8080":"password2"
      },
      "timeout":600,
      "method":"chacha20",
      "auth": true
      }
      ```
      
   * 编辑 supervisor 的配置文件：
   
      ```bash
      vi /etc/supervisor/conf.d/shadowsocks.conf
      ```
      
      *如果需要选定1024内的端口号，可能必须用root用户，即使用1024内的端口号，要使用root权限*
      
      ```JSON
      [program:shadowsocks]
      command=ssserver -c /etc/shadowsocks.json
      autorestart=true
      user=root
      ```
      
   * 现在即可通过下面的命令启动 shadowsocks 服务端，并检查其状态：
   
      ```bash
      supervisorctl reload
      supervisorctl status
      ```
      
   * 在 shadowsocks 客户端检查连接无误后再进入下一步。
   
3. kcptun 服务端的部署

   * 下载安装 kcptun：
   
      ```bash
      mkdir /root/kcptun
      cd /root/kcptun
      ln -sf /bin/bash /bin/sh
      wget https://github.com/xtaci/kcptun/releases/download/v20161118/kcptun-linux-amd64-20161118.tar.gz
      tar -zxf kcptun-linux-amd64-*.tar.gz
      ```
      
   * 配置 kcptun 启动和停止文件：
   
      *启动文件无需任何修改，稍后通过修改配置文件控制 kcptun 服务参数。*
      
      ```bash
      vi /root/kcptun/start.sh
      ```
      
      ```bash
      #!/bin/bash
      cd /root/kcptun/
      ./server_linux_amd64 -c /root/kcptun/server-config.json > kcptun.log 2>&1 &
      echo "Kcptun started."
      ```

      
      ```bash
      vi /root/kcptun/stop.sh
      ```
      
      ```bash
      #!/bin/bash
      echo "Stopping Kcptun..."
      PID=`ps -ef | grep server_linux_amd64 | grep -v grep | awk '{print $2}'`
      if [ "" !=  "$PID" ]; then
      echo "killing $PID"
      kill -9 $PID
      fi
      echo "Kcptun stoped."
      ```
      
   * 编辑 kcptun 的配置文件：
   
      ```bash
      vi /root/kcptun/server-config.json
      ```
   * 下面的配置文件中，需要注意以下参数：
   
      **listen** 参数为 kcptun 服务的端口； **target** 参数必须为本地 shadowsocks 使用的端口之一，也就是通过 kcptun 加速的端口； **key** 参数为 kcptun 使用的密码，自行设定。其他参数参考[项目介绍](https://github.com/xtaci/kcptun/blob/master/README.md)。
   
      ```JSON
      {
      "listen": ":443",
      "target": "127.0.0.1:1080",
      "key": "password3",
      "crypt": "salsa20",
      "mode": "fast",
      "mtu": 1350,
      "sndwnd": 1024,
      "rcvwnd": 1024,
      "datashard": 5,
      "parityshard": 5,
      "dscp": 46,
      "nocomp": true,
      "acknodelay": false,
      "nodelay": 0,
      "interval": 40,
      "resend": 0,
      "nc": 0,
      "sockbuf": 4194304,
      "keepalive": 10
      }
      ```

   * 现在可以通过`sh /root/kcptun/start.sh`命令启动 kcptun 服务，通过`sh /root/kcptun/stop.sh`停止。

   * 将 kcptun 服务设为开机启动：
   
      ```bash
      echo "sh /root/kcptun/start.sh" >> /etc/rc.local
      ```
      
4. kcptun 在 Windows 客户端上的启用与配置
   
   * 在[项目发布页](https://github.com/xtaci/kcptun/releases/)下载 Windows 版 kcptun 并解压缩。注意最好将 ***client_windows_amd64.exe*** 文件放在全英文路径的单独文件夹中。
   
   * 在上述文件夹中新建文本文档 `run.vbs`:

      ```vbs
      Dim RunKcptun
      Set fso = CreateObject("Scripting.FileSystemObject")
      Set WshShell = WScript.CreateObject("WScript.Shell")
      currentPath = fso.GetFile(Wscript.ScriptFullName).ParentFolder.Path & "\"
      configFile = currentPath & "client-config.json"
      logFile = currentPath & "kcptun.log"
      exeConfig = currentPath & "client_windows_amd64.exe -c " & configFile
      cmdLine = "cmd /c " & exeConfig & " > " & logFile & " 2>&1"
      WshShell.Run cmdLine, 0, False
      'WScript.Sleep 1000
      'Wscript.echo cmdLine
      Set WshShell = Nothing
      Set fso = Nothing
      WScript.quit
      ```
      
   * 在同一文件夹中新建客户端配置文件 `client-config.json`:

      注意配置文件中的以下参数必须与上面服务端的配置文件参数完全一致： *key*, *crypt*, *mode*, *mtu*, *datashard*, *parityshard*, *nocomp*。
      
      **localaddr** 参数为本地服务的端口号，亦即 shadowsocks 客户端中填写的端口号； **remoteaddr** 参数为 shadowsocks 服务器的 IP 地址和其 kcptun 服务的端口号。
      
      其他设置事项请参考[项目介绍](https://github.com/xtaci/kcptun/blob/master/README.md)。
      
      ```JSON
      {
      "localaddr": ":12345",
      "remoteaddr": "10.10.10.10:443",
      "key": "password3",
      "crypt": "salsa20",
      "mode": "fast",
      "conn": 1,
      "autoexpire": 60,
      "mtu": 1350,
      "sndwnd": 128,
      "rcvwnd": 1024,
      "datashard": 5,
      "parityshard": 5,
      "dscp": 46,
      "nocomp": true,
      "acknodelay": false,
      "nodelay": 0,
      "interval": 40,
      "resend": 0,
      "nc": 0,
      "sockbuf": 4194304,
      "keepalive": 10
      }
      ```
      
   * 新建`stop.bat`文件，用于停止 kcptun 客户端。
   
      ```bat
      taskkill /f /im client_windows_amd64.exe
      ```
      
   * 现在即可双击`run.vbs`运行 kcptun 服务，同时打开 shadowsocks 客户端，服务器地址填写 `localhost`，服务器端口为上述 kcptun 客户端配置文件中 *localaddr* 参数的数值，密码为服务器上 shadowsocks 配置文件中设置的密码，加密方式为 *CHACHA20*。
   
5. 在 shadowsocks Android 客户端配置 kcptun

   目前 [shadowsocks Android 客户端](https://play.google.com/store/apps/details?id=com.github.shadowsocks)已经集成了对 kcptun 的支持。
   
   * 在 shadowsocks Android 客户端的配置中**启用 KCP 协议**，在 **KCP端口** 中填写 kcptun 服务的端口号。
   
   * 在 **KCP 参数**中填写参数，**除最后一项外其他参数必须与服务端配置文件中的参数一致**。
   
      ```
      --key password3 --crypt salsa20 --mode fast --datashard 5 --parityshard 5 --nocomp --dscp 46
      ```
   
   * 启动 shadowsocks 客户端服务即可。
   
------
   
* 使用的项目的Github地址：

***shadowsocks***: https://github.com/shadowsocks/shadowsocks

***kcptun***: https://github.com/xtaci/kcptun

***supervisor***: https://github.com/Supervisor/supervisor

***libsodium***: https://github.com/jedisct1/libsodium

***shadowsocks Android***: https://github.com/shadowsocks/shadowsocks-android
