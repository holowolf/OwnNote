# Python常用

1. Python pip 升级全部包 `pip freeze --local | grep -v '^-e' | cut -d = -f 1  | xargs -n1 pip install -U`

2. pip 修改源

    ```txt
    指定root或用户目录下pip conf文件并修改源
    vim ~/.pip/pip.conf

    [global]
    index-url = https://mirrors.aliyun.com/pypi/simple/
    [install]
    trusted-host=mirrors.aliyun.com
    ```

3. ip代理池

    ```txt
    https://github.com/jhao104/proxy_pool
    ```

