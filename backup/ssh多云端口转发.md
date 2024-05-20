代理腾讯云端口：  ssh -fN tencent_port_proxy

![image](https://github.com/randolph555/blog/assets/22696026/cc25676d-5969-4dfa-816b-c85b1f8b5ec1)


查看后台挂起服务

![image](https://github.com/randolph555/blog/assets/22696026/ad98fe4e-7138-4691-a658-92ef1d50ce67)


查看端口是否监听
![image](https://github.com/randolph555/blog/assets/22696026/56d5e447-42f8-403f-b88f-580e2f152565)


验证是否可以连接
mysql -A -h 127.0.0.1 -P 3306 -u superset -p