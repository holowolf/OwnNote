#!/bin/bash
sudo apt-get update && sudo apt-get install -y      apt-transport-https      ca-certificates      curl      gnupg2      lsb-release      software-properties-common


### ustc源
curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/debian/gpg | sudo apt-key add -


### 添加docker-ce源
sudo add-apt-repository    "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/debian    stretch    stable"
