# PS 自动恢复不生效：明明打开了，崩溃后为什么还是找不回文件？

记录日期：2026-11-02
验证环境：Windows 26H2（构建 26340.9843），Photoshop 2026 27.8.0.13，E: 为唯一暂存盘。

我的 PS 一直开着自动恢复，但崩溃后从来没有恢复过文件。同一台电脑上的 Illustrator 自动恢复正常。最后查到，PS 正在尝试写入一个非法路径：E:\PSAutoRecover\E:。把 Windows“文档”入口调整到一个有正常父目录名称的位置后，恢复快照开始生成，真实崩溃后也成功找回了未保存的修改。

这个案例适合重点检查三件事同时出现的环境：非 C 盘作为暂存盘；Windows“文档”移到了类似 E:\Document 的盘根下一级目录；PSAutoRecover 长期为空。真正有判断力的证据，是文件访问日志中出现 NAME INVALID。

## 从“设置开着”到找到失败的那一次写入

最初做过重新开关自动恢复、重建 PSAutoRecover、重置偏好、隔离插件等检查，都没有解决。恢复文件夹出现了，里面仍然没有快照。

知乎作者 vekia 提到恢复位置和暂存盘，Adobe 社区也有人报告“C: 不是第一暂存盘时恢复失效”。这给了一个有效的对照方向：临时用 C:，本机生成了恢复快照，异常退出后恢复成功；回到 E:，再次失效。[1][2]

随后用 Microsoft Process Monitor 查看 Photoshop.exe 对恢复目录的访问，捕获到了这条记录：

    进程：Photoshop.exe
    路径：E:\PSAutoRecover\E:
    结果：NAME INVALID
    状态码：0xC0000033

盘符开头的 E: 合法；拼在目录中间作为一个子目录名的 E:，在这里却成了非法路径。因此反复重建一个空的 PSAutoRecover 文件夹，并没有解决真正失败的那次目录访问。

## 为何会拼出这个路径

本机 Windows“文档”的位置是 E:\Document。我只调整了“文档”的路径入口，PS 尝试创建的恢复目录名就随之改变。

根据这一前后变化推断，PS 在这条恢复逻辑中可能取了“文档”父目录的名称，作为恢复路径的一部分。文档直接位于盘根下一级时，取出来的是 E:。这是本机行为证据支持的解释，尚待其他环境独立复现。

## 实际修复：保留原文件，用目录联接提供兼容入口

本机最终路径如下：

    原文档内容：E:\Document
    新文档入口：E:\PSFolderCompat\Documents
    目录联接：E:\PSFolderCompat\Documents → E:\Document
    新恢复目录：E:\PSAutoRecover\PSFolderCompat

原文档内容没有搬迁。E:\Document 与新的入口都通向原来的文件。

具体做法是先保存工作、退出 PS，备份 Windows“文档”的路径设置，然后创建上述目录联接，并把当前用户以下两处 Personal 值指向新入口，保留各自原有的注册表值类型：

    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders
    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders

本机 User Shell Folders 的 Personal 是 REG_EXPAND_SZ，Shell Folders 的 Personal 是 REG_SZ。修改前的值和类型都单独保留了备份。

这会改变 Windows“文档”的逻辑地址，其他使用该已知文件夹的软件也可能看到新入口。因此实际操作应对应自己的文档位置，先核对目录联接目标，并保存原值用于回退。目录联接要随修复保留，清理时应注意它通向实际文档内容。

以上是本机路径对照和操作记录。其他电脑应先确认同类 NAME INVALID 记录，再决定是否采用这种兼容方式。

## 怎样确认真的修好了

我把自动恢复间隔设为 5 分钟，暂存盘继续使用 E:，用专门的测试 PSD 验证：

1. 保存测试文档，再移动一个图层，留下明确的未保存变化。
2. 等待恢复间隔，看到 E:\PSAutoRecover\PSFolderCompat 中生成新的 .psb，大小与修改时间都有效。
3. 本轮随后发生了正在排查的网络变化崩溃，重启 PS 自动打开“恢复的”文档。
4. 对比崩溃前后，图层名称及坐标完全一致，包括尚未保存的移动操作。
5. 后续做完网络兼容试验，又验证了一次“生成新快照—只强制结束测试会话—重启恢复”，结果一致。

第一份成功快照在 17:05 生成，大小 441200 字节；17:06 真实崩溃，17:07 自动恢复。后续 17:20 生成新快照，17:21 的异常退出验证也通过。这里的时间均为 2026-11-02、Asia/Shanghai。

自动恢复保留的是最近一次成功写出的快照。普通关闭文档并选择放弃修改，会走正常清理流程，适合用其他专门的测试方式验证崩溃恢复。

## 给遇到相同问题的人

先确认“自动存储恢复信息”开启、间隔合理、暂存盘有空间，再看恢复文件是否真的生成。如果一直为空，可以用 Process Monitor 设置进程过滤 Photoshop.exe、路径包含 PSAutoRecover，重点看实际访问路径和失败结果。[3]

我的 Windows“文档”位置、非法路径和前后修复结果，是这次最有价值的线索。将 C: 临时作为对照有助于缩小范围；最终结果也说明，这台电脑可以继续用 E: 暂存盘并正常自动恢复。

本记录来自一台现有安装环境，包含此前已有的本地程序修改；尚未完成全新官方安装上的自动恢复对照。版本号、文档位置和实际错误记录应一起判断。文字编辑卡死仍是另一项待排查问题。

## 来源与致谢

[1] vekia 的知乎回答，为暂存盘和恢复目录的排查提供了线索：
https://www.zhihu.com/question/562919083/answer/3542219799

[2] Adobe 社区 Auto recover still not working when C: is not the scratch disk：
https://community.adobe.com/questions-712/auto-recover-still-not-working-when-c-is-not-the-scratch-disk-1082587

[3] Microsoft Process Monitor 官方说明：
https://learn.microsoft.com/en-us/sysinternals/downloads/procmon

用于检索的术语：Photoshop 自动恢复不生效；自动保存；PSAutoRecover 空文件夹；非 C 盘暂存盘；文档目录迁移；NAME INVALID；Photoshop AutoRecover empty；Documents known folder。
