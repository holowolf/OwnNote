在PHPStorm中设置部署
在PHPStorm中，转到Settings -> Build, Execution, Deployment添加新的SFTP部署。设置如下：

SFTP主机：$ {FOUND_IP} - 例如172.17.0.1
港口：2222（可在laradock .env中找到 - ）
根路径：/ var / www
用户名：root
验证类型：密钥对
私钥文件： ${LARADOCK_FOLDER}/workspace/insecure_id_rsa
单击测试SFTP连接 - 应该工作。
在PHPStorm中设置远程解释器
去 Settings -> Languages & Frameworks -> PHP
单击CLI Interpreter旁边的三个点
添加一个新的，选择从docker，vagrant，remote，..
选择部署配置 - 并选择以前配置的部署
单击确定