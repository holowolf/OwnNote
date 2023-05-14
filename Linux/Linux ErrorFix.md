
-bash: warning: setlocale: LC_ALL: cannot change locale (zh_CN.utf8)

sudo dpkg-reconfigure locales

添加zh-cn语言修复


sudo echo "blacklist pcspkr" >> /etc/modprobe.d/blacklist

rmmod pcspkr

ristretto 图片查看工具
htop 数据显示工具
xfce4-datetime-plugin  xfce 时钟工具
xfce4-genmon-plugin 自定义输出插件(神器)
xfce4-cpugraph-plugin cpu显示插件



https://github.com/Genymobile/gnirehtet
让手机使用电脑的网络进行连接, 可用于电脑抓包显示.


将当前用户加入 wireshark 组 即可有权限使用 /usr/bin/dumpcap
sudo gpasswd -a ${USER} wireshark