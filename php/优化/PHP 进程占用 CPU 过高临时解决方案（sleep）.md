# PHP 进程占用 CPU 过高临时解决方案（sleep）

### 1、背景

EB 系统很多与平台相关的自动任务在执行相关逻辑时，由于很多有涉及请求外部平台接口，且循环处理数据的逻辑，导致进程一直在占用 CPU 时间片，从而使得别的进程无法抢占切换 CPU 时间片，导致前端页面无法流畅访问，页面卡顿的问题，由此需要解决 PHP 进程一直占用 CPU 的问题。



### 2、解决方案

在大量循环处理数据、执行外部请求的时候先检测系统当前占用的 CPU 百分比，如果超过一定的比例，则当前进程先 sleep 一段时间，让出执行 CPU 时间片，使得其他进程能快速进行切换执行。

具体逻辑就是，使用命令行 ps aux | grep '.*/data/www/.*\.php$' | awk '{sum +=$3}END{print sum}'

检测所有的自动服务占用 cpu 总和，再除以 CPU 核心数，当大于指定的比例时，进行 usleep，代码如下：

```
    // Common_Common::checkCpuUsageAndSleep();
    /**
     * @desc 检查自动服务使用 cpu 占比，超过一定百分比时，休眠一定时间
     * @author Huangbin <huangbin2018@qq.com>
     * @param int $maxUsage 最大 cpu 使用量（单核即可）
     * @param int $uSec 休眠指定时间（毫秒）
     * @return bool
     */
    public static function checkCpuUsageAndSleep($maxUsage = 80, $uSec = 200, $checkConfig = false)
    {
        static $sapiType = '';
        if ($sapiType == '') {
            $sapiType = php_sapi_name();
        }
        if (substr($sapiType, 0, 3) != 'cli') {
            return false;
        }
        // 当检查配置时，需要配置 CHECK_AUTO_RUN_CPU_USAGE_SLEEP 为 1 才有效
        if ($checkConfig) {
            if (!defined('CHECK_AUTO_RUN_CPU_USAGE_SLEEP') || CHECK_AUTO_RUN_CPU_USAGE_SLEEP != 1) {
                return false;
            }
        }
        static $coreNums = 0;
        if ($coreNums == 0) {
            // cpu 核心数
            preg_match_all('/^processor/m', file_get_contents('/proc/cpuinfo'), $matches);
            $coreNums = count($matches[0]);
        }
        // 自动任务占用 CPU 总数
        $cmd = "ps aux | grep '.*/data/www/.*\.php$' | awk '{sum +=$3}END{print sum}'";
        $output = shell_exec($cmd);
        if ($output) {
            $output = trim($output);
            $usage = $output;
            $uSec = (int)$uSec;
            $sleepRate = 1000 * $uSec; // 默认 200 毫秒
            if ($usage >= ($maxUsage * $coreNums)) {
                // 当前自动任务使用 cpu 总百分比超出时，sleep 一下
                usleep($sleepRate);
            }
        }
        
        return true;
    } 
```