<!-- TOC -->

- [什么是`Lunux Cgroups`](#01-什么是`LunuxCgroups`)
	- [`Cgroups`中的四个组件](#011-`Cgroups`中的四个组件)
	- [四个组件的相互关系](#012-四个组件的相互关系)
	- [`Kernel`接口](#013-`Kernel`接口)
	- [用`Go`实现通过`cgroup`限制容器资源](#014-用`Go`实现通过`cgroup`限制容器资源)
		- [技术总结](#0141-技术总结)
		- [参考:](#0142-参考:)

<!-- /TOC -->

---------------------------------

<a id="toc_anchor" name="#01-什么是`LunuxCgroups`"></a>

## 0.1. 什么是`Lunux Cgroups`

`Lunux Cgroups` 提供了对一组进程及子进程的资源限制.控制和统计的能力,这些资源包括硬件资源`CPU`,`Memory`,`DIsk`,`Network`等等

<a id="toc_anchor" name="#011-`Cgroups`中的四个组件"></a>

### 0.1.1. `Cgroups`中的四个组件

* `Cgroup`((控制组) 是对进程分组管理的一种机制,一个`Cgroup`包含一组进程,并可以在上面添加添加`Linux Subsystem`的各种参数配置,将一组进程和一组`Subsystem`的系统参数关联起来

* `Subsystem`(子系统)是一个资源调度控制器不同版本的`Kernel`所支持的有所偏差,可以通过`cat /proc/cgroups` 查看
    * `blkio` 对块设备(比如硬盘)的IO进行访问限制
    * `cpu` 设置进程的`CPU`调度的策略,比如`CPU`时间片的分配
    * `cpuacct` 统计/生成`cgroup`中的任务占用CPU资源报告
    * `cpuset` 在多核机器上分配给任务(`task`)独立的`CPU`和内存节点(内存仅使用于`NUMA`架构)
    * `devices` 控制`cgroup`中对设备的访问
    * `freezer` 挂起(suspend) / 恢复 (resume)`cgroup` 中的进程
    * `memory` 用于控制`cgroup`中进程的占用以及生成内存占用报告
    * `net_cls` 使用等级识别符（classid）标记网络数据包，这让 Linux 流量控制器 **tc** (`traffic controller`) 可以识别来自特定 cgroup 的包并做限流或监控
    * `net_prio` 设置`cgroup`中进程产生的网络流量的优先级
    * `hugetlb` 限制使用的内存页数量
    * `pids` 限制任务的数量
    * `ns` 可以使不同`cgroups`下面的进程使用不同的`namespace`.
    每个`subsystem`会关联到定义的`cgroup`上,并对这个`cgoup`中的进程做相应的限制和控制.

* `hierarchy`树形结构的 `CGroup` 层级，每个子 `CGroup` 节点会继承父 `CGroup` 节点的子系统配置，每个 `Hierarchy` 在初始化时会有默认的 `CGroup`(`Root CGroup`)


比如一组`task`进程通过`cgroup1`限制了`CPU`使用率,然后其中一个日志进程还需要限制磁盘IO,为了避免限制磁盘IO影响到其他进程,就可以创建`cgroup2`,使其继承`cgroup1`并限制磁盘IO,这样这样`cgroup2`便继承了`cgroup1`中对CPU使用率的限制并且添加了磁盘IO的限制而不影响到`cgroup1`中的其他进程

* `Task` (任务) 在`cgroups`中,任务就是系统的一个进程


<a id="toc_anchor" name="#012-四个组件的相互关系"></a>

### 0.1.2. 四个组件的相互关系

1. 系统在创建新的`hierarchy`之后,该系统的所有任务都会加入这个`hierarchy`的`cgroup`---称之为`root cgroup`,此`cgroup`在创建`hierarchy`自动创建,后面在该`hierarchy`中创建都是`cgroup`根节点的子节点

2. 一个`subsystem`只能附加到一个`hierarchy`上面

3. 一个`hierarchy`可以附加多个`subsystem`

4. 一个`task`可以是多个`cgroup`的成员,但这些`cgroup`必须在不同的`hierarchy` 

5. 一个进程`fork`出子进程时,该子进程默认自动成为父进程所在的`cgroup`的成员,也可以根据情况将其移动到到不同的`cgroup`中.

![](./res/mdImg/CGroups.png)

图 1. CGroup 层级图 (来源:https://www.ibm.com/developerworks/cn/linux/1506_cgroup/index.html)



---------------------------------

<a id="toc_anchor" name="#013-`Kernel`接口"></a>

### 0.1.3. `Kernel`接口

`Cgroups`中的`hierarchy`是一种树状组织结构,`Kernel`为了使对`Cgroups`的配置更加直观,是通过一个虚拟文件系统来配置`Cgroups`的,通过层级虚拟出`cgroup`树,例子操作如下

1. 创建并挂载一个`hierarchy`(`cgroup`)树

```Bash
# 创建一个`hierarchy`
root@DESKTOP-UMENNVI:~# mkdir -p cgroup-test/

# 挂载一个hierarchy
root@DESKTOP-UMENNVI:~# sudo mount -t cgroup -o none,name=cgroup-test cgroup-test ./cgroup-test/

# 挂载后我们就可以看到系统在这个目录下生成了一些默认文件
root@DESKTOP-UMENNVI:~# ls ./cgroup-test/
cgroup.clone_children  cgroup.procs  cgroup.sane_behavior  notify_on_release  release_agent  tasks
```

这些文件就是`hierarchy`中`cgroup`根节点的配置项,这些文件的含义是

* `cgroup.clone_children` `cpuset`的`subsystem`会读取这个配置文件,如果这个值(默认值是0)是 **1** 子`cgroup`才会继承父`cgroup`的`cpuset`的配置

* `cgroup.procs`是树中当前节点`cgroup`中的进程组ID,现在的位置是根节点,这个文件中会有现在系统中所有进程组的ID (查看目前全部进程PID `ps -ef | awk '{print $2}'`)

* `notify_on_release`和`release_agent` 会一起使用
    * `notify_on_release` 标志当这个`cgroup`最后一个进程退出的时候是否执行了`release_agent`

    * `release_agent` 则是一个路径,通常用作进程退出后自动清理不再使用的`cgroup`

* `task` 标识该`cgroup`下面进程ID,如果把一个进程ID写到`task`文件中,便会把相应的进程加入到这个`cgroup`中

2. 在刚刚创建好的`hierarchy`上`cgroup`根节点中扩展出两个子`cgroup`

```Bash
## 进入到刚刚创建的 hierarchy 内
root@DESKTOP-UMENNVI:~# cd cgroup-test/
root@DESKTOP-UMENNVI:~/cgroup-test# mkdir  -p cgroup-1 cgroup-2

## 查看目录树
root@DESKTOP-UMENNVI:~/cgroup-test# tree
.
├── cgroup-1
│   ├── cgroup.clone_children
│   ├── cgroup.procs
│   ├── notify_on_release
│   └── tasks
├── cgroup-2
│   ├── cgroup.clone_children
│   ├── cgroup.procs
│   ├── notify_on_release
│   └── tasks
├── cgroup.clone_children
├── cgroup.procs
├── cgroup.sane_behavior
├── notify_on_release
├── release_agent
└── tasks

2 directories, 14 files
```

可以看到,在一个`cgroup`的目录下创建文件夹时,`Kernel`会把文件夹标记记为子`cgroup`,它们会继承父`cgroup`的属性


3. 在`cgroup`中添加和移动进程
一个进程在一个`hierarchy`中,只能在一个`cgroup`节点上存在,系统的所有进程都默认在`root cgroups`上,我们可以将进程移动到其他的`cgroup`节点,只需要将进程ID移动到其他`cgroup`节点的`tasks`文件中即可

```Bash
## 进入到 cgroup-1
root@DESKTOP-UMENNVI:~/cgroup-test# cd cgroup-1/

## 显示当前终端PID
root@DESKTOP-UMENNVI:~/cgroup-test/cgroup-1# echo $$
1945

## 将本终端移动到 cgroup-1
root@DESKTOP-UMENNVI:~/cgroup-test/cgroup-1# sudo sh -c "echo $$ >> tasks"

## 检查进程所处 cgroup
root@DESKTOP-UMENNVI:~/cgroup-test/cgroup-1# cat /proc/1945/cgroup 
14:name=cgroup-test:/cgroup-1
13:rdma:/
12:pids:/
11:hugetlb:/
10:net_prio:/
9:perf_event:/
8:net_cls:/
7:freezer:/
6:devices:/
5:memory:/
4:blkio:/
3:cpuacct:/
2:cpu:/
1:cpuset:/
0::/
root@DESKTOP-UMENNVI:~/cgroup-test/cgroup-1# 
```

可以看到当前的`1945`进程已经被加到`cgroup-test:/cgroup-1`中了

4. 通过`subsystem`限制`cgroup`中进程的资源

在上面创建`hierarchy`的时候,这个`hierarchy`并没有关联到任何的`subsystem`,因此我们需要手动创建`subsystem`挂载到这个`cgroup-1`中

```Bash
## 挂载 memory subsystem 到 cgroup-test/cgroup-1/memory
root@DESKTOP-UMENNVI:~/cgroup-test/cgroup-1# mkdir -p memory

root@DESKTOP-UMENNVI:~/cgroup-test/cgroup-1# mount -t cgroup -o memory cgoup-1-mem ./memory

## 创建限制内存的 cgroup (limit-mem 可以替换成任意字符串)
root@DESKTOP-UMENNVI:~/cgroup-test/cgroup-1#  mkdir -p memory/limit-mem

## 将当前进程移动到这个 cgroup 中
root@DESKTOP-UMENNVI:~/cgroup-test/cgroup-1# echo $$ > memory/limit-mem/tasks

## 运行 stress 进程占用 200M 内存
root@DESKTOP-UMENNVI:~/cgroup-test/cgroup-1# stress --vm-bytes 200m --vm-keep -m 1
stress: info: [308] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd

## 结束进程
Ctrl+c / Command + c
```

开始限制 `cgroup` 内进程的内存使用量

```Bash
## 设置最大 cgroup 的最大内存占用为100MB
root@DESKTOP-UMENNVI:~/cgroup-test/cgroup-1# sudo echo 100M > memory/limit-mem/memory.limit_in_bytes

## 运行 stress 进程占用 200M 内存
stress --vm-bytes 200m --vm-keep -m 1

```

另起终端,查看`stress`进程占用内存情况

```Bash
## 此时查看进程ID号
root@DESKTOP-UMENNVI:~/cgroup-test/cgroup-1# ps -ef | grep stress | grep -v grep
root       496   258  0 11:14 pts/0    00:00:00 stress --vm-bytes 200m --vm-keep -m 1
root       497   496 23 11:14 pts/0    00:00:14 stress --vm-bytes 200m --vm-keep -m 1

## 可以看到有两个 stress 进程,其中有一个是 497 的子进程,我们需要查看的进程PID就是那个子进程

## 查看进程占用情况
root@DESKTOP-UMENNVI:~/cgroup-test/cgroup-1# cat /proc/497/status  | grep Vm
VmPeak:   213044 kB     # 进程所使用虚拟内存的峰值
VmSize:   213044 kB     # 进程当前使用的虚拟内存大小
VmLck:         0 kB     # 已经锁住的物理内存的大小
VmPin:         0 kB     # 进程所使用的物理内存峰值
VmHWM:    100292 kB     # 进程当前使用的物理内存的峰值
VmRSS:     99956 kB     # 进程当前使用的物理内存大小
VmData:   204996 kB     # 进程占用的数据段大小
VmStk:       132 kB     # 进程占用的栈大小
VmExe:        20 kB     # 进程占用的代码段大小(不包含链接库)
VmLib:      3764 kB     # 进程所加载的动态库所占用的内存大小(可能与其他进程共享)
VmPTE:       452 kB     # 进程占用的页表大小 (交换表项数量)
VmSwap:   105200 kB     # 进程所使用的交换区大小

```

可以看到`stress`进程实际物理内存占用只有 `99956kB` ,其余占用内存分配给了`swap`分区了,说明已经成功将进程最大(物理内存)占用限制到了 `100M`


----------------------------------

<a id="toc_anchor" name="#014-用`Go`实现通过`cgroup`限制容器资源"></a>

### 0.1.4. 用`Go`实现通过`cgroup`限制容器资源

下面在`Namespace`的基础上,加上`cgroup`限制实现一个`demo`,使其能够具有限制容器内存的功能

```Go
// Cgroup/limitMem/demo.go
package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"os"
	"os/exec"
	"path"
	"strconv"
	"syscall"
)

const cgroupMemoryHierarchyMount = "/sys/fs/cgroup/memory"
const limitMemory = "100M"

func main() {
	//-----------------------------------------------------
	// 5.运行 stress 进程测试内存占用
	if os.Args[0] == "/proc/self/exe" {
		//-----------------------------------------------------
		// 6. 挂载容器内的 /proc 的文件系统
		//Mount /proc to new root's  proc directory using MNT namespace
		if err := syscall.Mount("proc", "/proc", "proc", uintptr(syscall.MS_NOEXEC|syscall.MS_NOSUID|syscall.MS_NODEV), ""); err != nil {
			fmt.Println("Proc mount failed,Error : ", err)
		}

		// 7. 异步执行一个 sh 进程进入到容器内
		go func() {
			cmd := exec.Command("/bin/sh")

			cmd.SysProcAttr = &syscall.SysProcAttr{}

			cmd.Stdin = os.Stdin
			cmd.Stdout = os.Stdout
			cmd.Stderr = os.Stderr
			cmd.Run()
			os.Exit(1)
		}()

		// 8. 运行 stress 进程
		log.Printf("Current pid %d \n", syscall.SYS_GETPID)
		cmd := exec.Command("sh", "-c", `stress --vm-bytes 200m --vm-keep -m 1`)
		cmd.SysProcAttr = &syscall.SysProcAttr{}

		cmd.Stdin = os.Stdin
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr

		log.Print("Close the program, press input `exit` \n")
		if err := cmd.Run(); err != nil {
			log.Fatal(err)
		} else {
			log.Printf("Stress process pid : %d \n", cmd.Process.Pid)
		}
		os.Exit(1)

	}

	//-----------------------------------------------------
	// 1, 先创建一个外部进程
	cmd := exec.Command("/proc/self/exe")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS,
	}

	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Start(); err != nil {
		log.Fatal(err)
		os.Exit(1)
	}

	//-----------------------------------------------------
	// 2. 在挂载了memory subsysyem 下创建限制内存的cgroup
	memory_limit_path := path.Join(cgroupMemoryHierarchyMount, "memorylimit")
	if f, err := os.Stat(memory_limit_path); err == nil {
		if !f.IsDir() {
			if err = os.Mkdir(memory_limit_path, 0755); err != nil {
				log.Fatal(err)
			} else {
				log.Printf("Mkdir memory cgroup %s \n", path.Join(cgroupMemoryHierarchyMount, "memorylimit"))
			}
		}
	} else {
		if err = os.Mkdir(memory_limit_path, 0755); err != nil {
			log.Fatal(err)
		} else {
			log.Printf("Mkdir memory cgroup %s \n", path.Join(cgroupMemoryHierarchyMount, "memorylimit"))
		}
	}

	//-----------------------------------------------------
	// 3. 限制 cgroup 内进程最大物理内存<limitMemory>
	if err := ioutil.WriteFile(path.Join(memory_limit_path, "memory.limit_in_bytes"), []byte(limitMemory), 0644); err != nil {
		log.Fatal("Litmit memory error,", err)
	} else {
		log.Printf("Litmit memory %v sucessed\n", limitMemory)
	}

	log.Printf("Self process pid : %d \n", cmd.Process.Pid)

	//-----------------------------------------------------
	// 4. 将进程加入到 cgroup 中
	if err := ioutil.WriteFile(path.Join(memory_limit_path, "tasks"), []byte(strconv.Itoa(cmd.Process.Pid)), 0644); err != nil {
		log.Fatal("Move process to task error,", err)
	} else {
		log.Printf("Move process %d to task sucessed \n", cmd.Process.Pid)
	}

	cmd.Process.Wait()
}

```

运行程序

```Bash
root@DESKTOP-UMENNVI:# go run Cgroup/limitMem/demo.go 
2020/03/01 23:05:33 Litmit memory 100M sucessed
2020/03/01 23:05:33 Self process pid : 22761 
2020/03/01 23:05:33 Move process 22761 to task sucessed 
2020/03/01 23:05:33 Current pid 39 
2020/03/01 23:05:33 Close the program, press input `exit` 
# stress: info: [8] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
# 此时再按一下回车

# ps -ef      
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 23:05 pts/1    00:00:00 /proc/self/exe
root         6     1  0 23:05 pts/1    00:00:00 /bin/sh
root         7     1  0 23:05 pts/1    00:00:00 sh -c stress --vm-bytes 200m --vm-keep -m 1
root         8     7  0 23:05 pts/1    00:00:00 stress --vm-bytes 200m --vm-keep -m 1
root         9     8 36 23:05 pts/1    00:00:07 stress --vm-bytes 200m --vm-keep -m 1
root        10     6  0 23:05 pts/1    00:00:00 ps -ef

## 可以看到PID Namespace已经被隔离了,这里我们直接查看 stree 进程的内存占用
# cat /proc/9/status | grep Vm
VmPeak:   213044 kB
VmSize:   213044 kB
VmLck:         0 kB
VmPin:         0 kB
VmHWM:     98440 kB
VmRSS:     87220 kB
VmData:   204996 kB
VmStk:       132 kB
VmExe:        20 kB
VmLib:      3764 kB
VmPTE:       460 kB
VmSwap:   117936 kB

```

通过对`Cgroup`的配置,已经将容器中的`stress`进程的物理内存占用限制到了`100MB`

----------------------------------

<a id="toc_anchor" name="#0141-技术总结"></a>

#### 0.1.4.1. 技术总结
1. 在挂载了`memory subsystem`的`Hierarchy`上创建`cgroup`
2. 限制该`cgroup`的最大物理内存值
3. 将`fork`出来的进程加入到这个容器内
4. 在容器内重新挂载`/proc`使其跟宿主机隔离`PID Namespace`
5. 在容器内运行`stress`进程
6. 另起一个进程开`sh`进程进入到容器内



----------------------------------
<a id="toc_anchor" name="#0142-参考:"></a>

#### 0.1.4.2. 参考:

* https://www.ibm.com/developerworks/cn/linux/1506_cgroup/index.html



----------------------
[hierarchy]:data:image/png;base64,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