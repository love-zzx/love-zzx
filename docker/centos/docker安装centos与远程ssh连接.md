# 基于docker安装centos实现远程ssh连接

## 安装centos

1. 下载镜像：

   ```dockerfile
   docker pull centos
   ```

   

2. 启动镜像：

   ```shell
   docker run -itd --name os1 --privileged=true -p 10000:22 300e315adb2f  ./sbin/init
   ```

   

   ```wi
   $ docker run -itd --name os1 --privileged=true -p 10000:22 300e315adb2f  /sbin/init
   0d675f73207651c8ae4771af3be648c5d34cadb265cde41477f43cafe94d30dd
   docker: Error response from daemon: OCI runtime create failed: container_linux.go:367: starting container process caused: exec: "D:/soft/Git/usr/sbin/init": stat D:/soft/Git/usr/sbin/init: no such file or directory: unknown.
   
   这一步会报错，是因为找不到该目录，网上找了很久，只需要把加个.即可
   
   docker run -itd --name os1 --privileged=true -p 10000:22 300e315adb2f  ./sbin/init
   	解析：启动名为os1的容器，使root拥有真正的权限：privileged=true，端口映射：10000=>22 
   ```

3. 进入容器：

   ```
   **winpty docker exec -it os1 bash** 
   ```

   

4. ifconfig安装：

   ```
   yum install -y net-tools
   ```

   

5. service安装：

   ```
   yum install -y initscripts
   ```

   

6. ssh安装： 

   1. 查看是否已安装：

      ```
      sshd rpm -qa | grep ssh
      ```

      

   2. 安装ssh：

      ```
      **yum install -y openssh-server** 
      ```

      

   3. 启动ssh：

      ```
      service sshd restart
      ```

      

   4. 查看是否启动22端口：

      ```
      netstat -anpt |grep sshd
      ```

      

7. 开启远程连接ssh

   1. ```
      yum install -y vim passwd
      ```
   
      
   
   2. 开启远程登录权限：
   
      ```
      vim /etc/ssh/sshd_config #打开注释 PermitRootLogin yes, 允许密码登录,保存退出
      ```
   
       
   
   3. 设置root密码：
   
      ```
      passwd root
      ```
   
      
   
   4. 宿主机ssh远程登录：
   
      ```
      ssh root@127.0.0.1 -p 10000 或者使用xshell账号密码登录
      ```
   
      

