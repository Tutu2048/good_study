在完成vpn部署后，增加下述配置，提高vps的网络通信能力

1. **调整TCP窗口大小**

   - 编辑`/etc/sysctl.conf`文件，添加以下参数：

     复制

     ```
     net.ipv4.tcp_rmem = 4096 87380 16777216
     net.ipv4.tcp_wmem = 4096 65536 16777216
     ```

   - 应用配置：`sysctl -p`。

2. **开启TCP BBR拥塞控制算法**

   - 编辑`/etc/sysctl.conf`文件，添加以下参数：

     复制

     ```
     net.core.default_qdisc = fq
     net.ipv4.tcp_congestion_control = bbr
     ```

   - 应用配置：`sysctl -p`。

   - 重启系统以确保生效。