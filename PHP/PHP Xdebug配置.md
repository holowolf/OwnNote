## Xdebug配置

用的是xamp，直接php.ini里面的;zend_extension = "D:\xampp\php\ext\php_xdebug.dll"把注释去掉。
然后相关参数设定好，注意两个参数
xdebug.remote_autostart=On必须为On，要不Eclipse不会自动进入断点。
xdebug.collect_return=Off必须为Off，或者不设定，要不Thinkphp一直死循环在入口index.php
xdebug.auto_trace = On
xdebug.show_exception_trace = On
xdebug.profiler_append = 0
xdebug.profiler_enable = 1
xdebug.profiler_enable_trigger = 0
xdebug.profiler_output_dir = "D:\xampp\tmp"
xdebug.profiler_output_name = "cachegrind.out.%t-%s"
xdebug.remote_enable = On
xdebug.remote_autostart=On
xdebug.remote_handler = "dbgp"
xdebug.remote_host = "127.0.0.1"
xdebug.remote_port=9000
xdebug.trace_output_dir = "D:\xampp\tmp"
然后可以通过phpinfo.php看xdebug是否正常enable了