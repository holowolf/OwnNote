一.GoReplay简介
GoReplay 是一个简单且安全通过真实流量测试未上线产品的工具
GoReplay 工作方式：listener server 捕获流量，并将其发送至 replay server 或者保存至文件，replay server 会将流量转移至配置的地址。下图是GoReplay Workflow



二. GoReplay 输入输出参数
1. 输入参数
   --input-raw 捕获 HTTP 流量，需指定端口号
   --input-file 使用 --output-file 记录的文件作为输入
   --input-tcp 将多个 Gor 实例获取的流量聚集到一个 Gor 实例
   2.输出参数
   --output-http 相应流量的终端，接收 URL 地址
   --output-file 将获取的流量记录如文件
   --output-tcp 将获取的流量转移至另外的 Gor 实例，与 --input-tcp 组合使用
   --output-stdout debug 工具，将流量信息直接输出
   更多参数信息请参考goreplay帮助命令
   三. GoReplay 使用
1. 将 8888 端口流量输出到终端
   sudo ./goreplay --input-raw :8888 --output-stdout
2. 将 8888 端口流量镜像至 http://t1:8888
   sudo ./goreplay --input-raw :8888 --output-http "http://t1:8888"
3. 将 8888 端口流量写入文件
   sudo ./goreplay --input-raw :8888 --output-file requests.log
4. 流量限制
# 每秒最大 10 个请求数
sudo ./goreplay --input-raw :8888 --output-http "http://t1:8888|10"
# 获取少于 10% 的流量
sudo ./goreplay --input-raw :8888 --output-http "http://t1:8888|10%"
5.请求过滤（录制流量和回放流量都可使用）
--http-allow-url 允许正则 URL，不包换主句名部分
--http-disallow-url 不允许正则 URL
--http-allow-header 允许的 Header 头
--http-disallow-header 不允许的 Header 头
--http-allow-method 允许的请求方法
6.以长连接回放
--http-set-header "Connection: keep-alive"
7.加参数回放
--http-set-param "goreplay=1"
8. 按照录制流量的host进行回放
   -http-original-host
   9.绑定证书，模仿https回放
 --input-tcp-certificate "/root/goreplay/shein.com.cer" --input-tcp-certificate-key "/root/goreplay/shein.com.key" --output-tcp-secure
   四. 测试实例
   1.  测试背景
   黑五即将到来，这对商城的性能要求很高，所以想要通过对接口的压测，大概估算出商城可承受的压力以及瓶颈，但是通过压测工具发请求怕走缓存的比例不贴合实际，所以想录制下线上实际请求的数据，这样更贴合实际
2. 需要录制的请求
   /v2/product/prices/getPrices（批量获取商品价格）
3. 文件存放路径
   goreplay程序所在路径：/lingtian/opt/goreplay/goreplay
   录制数据存放路径：/lingtian/logs/goreplay/
4. 录制方式
   （1）按照请求api url录制
   将从 80 端口访问过来并且url匹配到/v2/product/prices/getPrices和/User/Wishlist/getLists关键字的流量写入到文件50.22.168.140_getPrices_requests.log
   /lingtian/opt/goreplay/goreplay --input-raw :80 --output-file /lingtian/logs/goreplay/50.22.168.140_getPrices_requests.log --http-allow-url  /v2/product/prices/getPrices  --http-allow-url /User/Wishlist/getLists
   （程序自动在文件后面加"_0"标识，如果已经存在"_0"的文件，再次新生成的文件程序会在后面加"_1"标识）
   （2）按照请求方法录制
   将从80端口访问过来的GET请求的流量写入到文件50.22.168.140_GET_method_requests.log
   /lingtian/opt/goreplay/goreplay --input-raw :80 --output-file /lingtian/logs/goreplay/50.22.168.140_GET_method_requests.log --http-allow-method GET 
5. 重放方式
                                                                                                                                                         （1）根据文件原速重放
                                                                                                                                                         /lingtian/opt/goreplay/goreplay --input-file "/lingtian/logs/goreplay/50.22.168.140_getPrices_requests_0.log" --output-http="http://10.28.75.55:80" -http-original-host
                                                                                                                                                         （2）按照2倍倍速重放，并指定head头host是 api.shein.com
                                                                                                                                                         /lingtian/opt/goreplay/goreplay --input-file "/lingtian/logs/goreplay/50.22.168.140_getPrices_requests_0.log|200%" --output-http="http://10.28.75.55:80" --http-set-header "Host: api.shein.com"
                                                                                                                                                         （3）按照原来倍速的50%回放
                                                                                                                                                         /lingtian/opt/goreplay/goreplay --input-file "/lingtian/logs/goreplay/50.22.168.140_getPrices_requests_0.log" --output-http="http://10.28.75.55:80|50%" --http-set-header "Host: api.shein.com"
                                                                                                                                                         （4）按照原来倍速的6倍、模仿https、带参数"goreplay=1"、以长连接方式回放，并且过滤掉UA带有google关键字的流量
                                                                                                                                                         /lingtian/opt/goreplay/goreplay --input-file "/lingtian/logs/goreplay/romwe_app/*.log|600%" --input-tcp-certificate "/tmp/goreplay/romwe.com.cer" --input-tcp-certificate-key "/tmp/goreplay/romwe.com.key" --output-tcp-secure -output-http "https://50.23.80.20:443" -http-original-host --http-set-header "Connection: keep-alive" --http-set-param "goreplay=1" -http-disallow-header 'User-Agent: (.*)Googlebot(.*)'
