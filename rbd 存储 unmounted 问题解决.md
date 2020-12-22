### rbd 存储 unmounted 问题解决

问题：服务器重启导致 harbor volume 挂载不上

1. 查看容器的状态

   `kubectl describe pod os-harbor-admreg-0`

   输出：

   ```
   Events:
     Type     Reason       Age                From                   Message
   
   ----     ------       ----               ----                   -------
   
     Normal   Scheduled    33m                default-scheduler      Successfully assigned default/os-harbor-admreg-0 to 5103p01-2-s2
     Warning  FailedMount  1m (x14 over 31m)  kubelet, 5103p01-2-s2  Unable to mount volumes for pod "os-harbor-admreg-0_default(aa4f0bca-3f42-11eb-8bb9-943bb0b77234)": timeout expired waiting for volumes to attach or mount for pod "default"/"os-harbor-admreg-0". list of unmounted volumes=[storage]. list of unattached volumes=[conf key config storage default-token-782l2]
     Warning  FailedMount  8s (x17 over 32m)  kubelet, 5103p01-2-s2  MountVolume.WaitForAttach failed for volume "harbor" : rbd image .blockDisk.rbd/j19707-5103p01-d039-harbor is still being used
   ```

2. 查看该 rbd 的信息

   `rbd info .blockDisk.rbd/j19707-5103p01-d039-harbor`

   输出：

   ```
   rbd image 'j19707-5103p01-d039-harbor':
           size 102400 MB in 25600 objects
           order 22 (4096 KB objects)
           data_pool: Pool
           block_name_prefix: rbd_data.5.9e157247efa259
           format: 2
           features: layering, deep-flatten, data-pool, thick-optimization-logic
           flags:
           create_timestamp: Thu Nov 26 19:08:44 2020
   ```

   继续查看是哪个客户端在使用 rbd：

   `rados listwatchers -p .blockDisk.rbd rbd_header.9e157247efa259`

   输出：

   ```
   watcher=172.25.16.16:0/1398331141 client.2178981554 cookie=140484085360912
   ```

3. 登录到这台机器，执行下面的命令

   `rbd nbd list`

   输出：

   ```
   pid   pool           image                      snap device
   44594 .blockDisk.rbd j19707-5103p01-d039-paas1  -    /dev/nbd0
   37946 .blockDisk.rbd j19707-5103p01-d039-harbor -    /dev/nbd3
   44525 .blockDisk.rbd j19707-5103p01-d039-paas1  -    /dev/nbd4
   ```

   使用 `rbd nbd unmap /dev/nbd3` unmap nbd，稍等几分钟，harbor 容器恢复正常。