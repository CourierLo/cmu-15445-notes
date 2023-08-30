# 高并发B+树
没什么好说的，按着B+树的定义写就行了，还有上锁规则除了latch crabbing之外，还要对根节点加一把锁，要先拿到这把锁才能够开始访问B+树，这是为了防止因为树的生长或下降导致的根节点发生变化，反正对于Insert和Delete而言，只要往下走时一直都无法发现安全节点，那么这把锁就不会放。不过注意这里加锁的仅仅是对从根节点往下访问的情形，而对于水平访问的迭代器则没有加锁(大坑)

画出B+树 dot -Tpng -O *.dot

打开coredump: `sudo systemctl enable apport.service`
关闭coredump: `sudo systemctl disable apport.service`

sudo sysctl -w kernel.core_pattern=/home/courierlo/bustub-private/build/core-%e.%p.%h.%t
systemctl restart apport

sudo echo "/home/courierlo/bustub-private/build/core-%e.%p.%h.%t" > /proc/sys/kernel/core_pattern 没用

如果不关闭sudo service apport stop，coredump位置：`/var/lib/apport/coredump`
否则就会在当前文件夹出现了

gdb指令
l 查看代码
thread(t) apply all bt
bt 查看栈帧
frame(f) n 切换栈帧， up down 上下切换栈帧
info locals 查看变量信息   p xx 打印变量

1. 查看进程：info inferiors
2. 查看线程：info threads
3. 查看线程栈结构：bt
4. 切换线程：thread n（n代表第几个线程）

查看程序的进程号
ps aux | grep program_name
查看进程的序列号
pstree -p `pid`

gdb
attach 进程号

ulimit -c unlimited