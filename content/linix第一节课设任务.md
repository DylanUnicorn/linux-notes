![[Pasted image 20260322215931.png]]

![[Pasted image 20260322220429.png]]
![[Pasted image 20260322220507.png]]
![[Pasted image 20260322225801.png]]

在脚本的最后，通常会调用 `exec run-init` ，将内存中的临时根目录切换到真正的物理硬盘上，并执行硬盘上的 `/sbin/init`（即 systemd），从而进入带桌面的 Ubuntu。


成功的截图：
![[Pasted image 20260323011758.png]]
