# PS 开了自动恢复还是丢文件？放着不用就闪退怎么办？
Photoshop AutoRecover not working and crashes when idle

记录日期：2026-11-02

明明开了 Photoshop 自动恢复，崩溃后还是找不回文件；电脑放着不用，PS 却自己退出，断网重连也会触发闪退。这里记录这两个问题在自己电脑上的排查过程、处理方法和实测结果。

验证环境：Windows 26H2（构建 26340.9843），Photoshop 2026 27.8.0.13。自动恢复已通过崩溃恢复验证；网络问题的局部修正通过六轮重连，长期待机和睡眠唤醒仍待观察。


## 阅读入口

| 案例 | 中文 | English |
| --- | --- | --- |
| 开了自动恢复，崩溃后仍丢文件 | [完整案例](docs/autorecover.zh-CN.md) | [Case report](docs/autorecover.en.md) |
| 放着不用、断网重连时闪退 | [完整案例](docs/network-crash.zh-CN.md) | [Case report](docs/network-crash.en.md) |

[脱敏验证摘要](evidence/2026-11-02.json) · [更新记录](CHANGELOG.md)

## 案例一：自动恢复不生效，PSAutoRecover 为空

现象：自动存储恢复信息开启，暂存盘为 E:，崩溃后没有恢复文档；同机 Illustrator 恢复正常。

直接证据：Photoshop 访问 E:\PSAutoRecover\E:，返回 NAME INVALID（0xC0000033）。Windows“文档”的原路径为 E:\Document。

处理：通过 E:\PSFolderCompat\Documents → E:\Document 的目录联接提供文档入口，修改当前用户两处 Personal 路径值。原文件保持原位，暂存盘继续使用 E:。

结果：新快照写入 E:\PSAutoRecover\PSFolderCompat。真实崩溃及后续测试会话的强制退出后，均自动恢复，未保存图层坐标一致。

推断：恢复目录中的某一组件可能来自 Documents 的父目录名称。盘根下的 Documents 位置与此异常路径有关；其他安装仍需独立复现。

## 案例二：网络变化触发 Photoshop 退出

现象：网络重连可触发退出；错误代码 0xe06d7363。转储包含 netprofm、GetConnectivity、COM/RPC 重入及 _Throw_Cpp_error。

本机观察：相关网络对象持有非递归锁时查询连接状态，COM 回调再次进入相同路径；本机异常实参 5 对应 RESOURCE_DEADLOCK_WOULD_OCCUR。

干预：完整备份后，对确切二进制中的一处网络锁初始化做局部兼容修改，允许同线程递归锁定。

结果：同一会话五轮重连及重启后的一轮重连均通过，PS 保持响应；恢复功能和正常退出也通过。

范围：六次短期回归不等于长期待机或睡眠唤醒的全面验证。文字编辑卡死仍待独立排查。

## 环境与证据范围

这是单机案例，安装在排查前已有本地程序修改。自动恢复尚未在完全干净的官方安装中复现，网络干预依赖确切二进制。阅读时应同时核对环境、错误路径和调用栈。

附带 [evidence-summary.json](evidence/2026-11-02.json) 为从本机记录中提取的脱敏结果摘要。摘要保留版本、时间、测试结果、示例路径和坐标核对；原始内存转储、完整文件访问轨迹及工作文档留在本机。

## 反馈时有用的信息

- Photoshop 与 Windows 的完整版本。
- 实际症状，以及网络变化或异常退出的具体触发方式。
- 暂存盘和 Documents 已知文件夹的位置；私人目录名称可替换后说明结构。
- PSAutoRecover 的实际失败路径、返回码，或相关崩溃栈摘要。
- 修改前与修改后的验证结果。

## English summary

These are two local case studies from Photoshop 27.8.0.13 on Windows 26H2, OS build 26340.9843, using an existing installation with pre-existing binary modifications.

AutoRecover: Photoshop attempted E:\PSAutoRecover\E: and received NAME INVALID while the Documents known folder was E:\Document. Providing a deeper Documents alias through a directory junction changed the recovery path to E:\PSAutoRecover\PSFolderCompat. The physical documents and E: scratch disk stayed in place. Recovery of the exact unsaved layer state passed after an actual crash and a later fixture-only forced exit.

Network crash: a reentrant COM GetConnectivity path attempted to lock a non-recursive ReachabilityImpl mutex again. The local error argument was 5, RESOURCE_DEADLOCK_WOULD_OCCUR. An exact-binary mutex compatibility experiment passed six reconnect trials, including one after application restart. This is a local finding with limited regression coverage; clean-install reproduction, real sleep/resume and long-duration behavior require further validation.

## References

Original Chinese recovery-directory discussion:
https://www.zhihu.com/question/562919083/answer/3542219799

Earlier non-C scratch-disk recovery discussion:
https://community.adobe.com/questions-712/auto-recover-still-not-working-when-c-is-not-the-scratch-disk-1082587

Related network/wake crash report:
https://community.adobe.com/bug-reports-711/photoshop-crashes-on-pc-wake-1641867

Microsoft Process Monitor:
https://learn.microsoft.com/en-us/sysinternals/downloads/procmon

Microsoft STL mutex implementation:
https://raw.githubusercontent.com/microsoft/STL/main/stl/src/mutex.cpp
https://raw.githubusercontent.com/microsoft/STL/main/stl/inc/xthreads.h
