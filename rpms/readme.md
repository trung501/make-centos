# 安装glibc时需要这样，注意不能降级，否则系统会出问题的
## 该镜像使用的是CentOS-7-x86_64-Minimal-2009.iso 
## 使用更高版本可以删除glibc相关包
sudo rpm -ivh glibc-2.22.90-21.el7.x86_64.rpm glibc-common-2.22.90-21.el7.x86_64.rpm --replacefiles
