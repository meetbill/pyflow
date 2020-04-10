## pyFlow - 一个轻量级的并行任务引擎
Chris Saunders（csaunders@illumina.com）版本：
$ {VERSION}


<!-- vim-markdown-toc GFM -->

* [摘要](#摘要)
* [特征](#特征)
* [安装](#安装)
* [编写 WORKFLOWS：](#编写-workflows)
    * [使用 WORKFLOWS](#使用-workflows)
    * [Logging:](#logging)
    * [State：](#state)
    * [其他：](#其他)
        * [电子邮件通知：](#电子邮件通知)
        * [Graph 输出：](#graph-输出)
        * [网站配置：](#网站配置)

<!-- vim-markdown-toc -->

## 摘要
pyFlow 在任务依赖关系图的上下文中管理正在运行的任务。它有一些相似之处。pyFlow 不是一个程序 , 它是一个 python 模块，通过使用 pyFlow API 编写常规 python 代码，使用 pyFlow 定义工作流。

## 特征

- 将工作流定义为 python 代码
- 在 localhost 或 sge(Sun Grid Engine 集群) 上运行工作流
- 继续部分完成的工作流程
- 任务资源管理：指定每个任务所需的线程数和内存数。
- 递归工作流规范：获取任何现有的 pyFlow 对象，并将其用作另一个 pyFlow 中的任务。
- 动态工作流规范：
  - 定义等待任务规范而不仅仅是任务，以便可以根据上游任务的结果定义任务（注意：递归工作流是更好的方法）
  - 可以在工作流程期间根据其他任务的结果取消任务 / 任务树
- 使用一致的工作流级别日志记录检测并报告所有失败的任务。
- 任务级日志记录：记录和修饰所有任务 stderr
  - 例如。时间/主机/[workflow_run]/taskid
- 任务计时：任务包装器功能为每个任务提供挂起时间
- 任务优先级：同时有资格运行的任务可以分配相对优先级，以便首先运行或排队。
- 任务互斥集：定义访问独占资源的任务集
- 有关作业完成 / 错误 / 异常的电子邮件通知
- 按指定的时间间隔提供正在进行的任务
- 以 dot 格式输出任务图

## 安装
pyFlow 可以在 2.4 到 2.7 系列的 python 版本上安装和使用

pyflow 模块可以使用标准的 python distutils 安装来安装。为此，请解压缩 tarball 并使用安装脚本，如下所示：

```bash
$ tar -xzf pyflow-XYZtar.gz
$ cd pyflow-XYZ
$ python setup.py build install
```
如果安装不方便，只需将 pyflow src / 目录添加到系统搜索路径即可。例如：

usepyflow.py：
```python
import sys
sys.path.append("/path/to/pyflow/src")

from pyflow import WorkflowRunner
```

## 编写 WORKFLOWS：

简而言之，pyFlow 工作流是通过创建一个继承自 pyflow.WorkflowRunner 的新类来编写的。然后，该类通过重载 WorkflowRunner.workflow（）方法来定义其工作流。通过实例化工作流类并调用 WorkflowRunner.run（）方法来运行工作流。

目录中提供了上述最小工作流设置和运行的非常简单的演示： ${pyflowDir}/demo/helloWorld/

还有其他几个演示工作流程： ${pyflowDir}/demo/simpleDemo- 基本功能沙箱 ${pyflowDir}/demo/subWorkflow- 显示递归工作流调用的工作原理

可以通过运行 ${pyflowDir}/doc/getApiDoc.py 或生成 pyflow API 的开发人员文档 python ${pyflowDir}/src/pydoc.py

bclToBam 转换的高级概念验证演示也可用于 ${pyflowDir}/demo/bclToBwaBam

### 使用 WORKFLOWS
运行 pyFlow 工作流时，所有日志和状态信息都写入单个“pyflow.data”目录。此目录的根目录在 workflow.run（）调用中指定。

### Logging:

pyFlow 创建主工作流级日志，2 个日志文件分别捕获所有任务 stdout 和 stderr。

工作流级日志信息将复制到 stderr 和 pyflow.data/logs/pyflow_log.txt。所有工作流日志消息都以“[time] [hosname] [workflow_run] [component]”为前缀。哪里：

- 'time'是 ISO 8601 格式的 UTC。
- 'workflow_run'是一个对于每次运行工作流而言都很弱的 ID。它由（1）run() PID 和（2）由同一进程在工作流上调用 run() 的次数组成。这两个值由下划线连接
- 'component' - pyflow 线程的名称，主线程是运行 worklow（）方法的'WorkflowManager'，以及轮询任务图并启动作业的'TaskManager'。

在任务日志中，仅装饰 stderr 流。这种情况下的前缀是：“[time] [hostname] [workflow_run] [taskname]”。'taskname'通常是在 addTask（）调用中为每个任务提供的标签。所有任务都由任务包装函数启动，来自 taskWrapper 的任何消息（与任务命令本身相反）将使用扩展的任务名： “pyflowTaskWrapper：$ {tasklabel}”。任务包装器写入日志的一个示例是报告其任务的总运行时间。

所有日志记录仅附加 - 即使在多次运行中，pyFlow 也不会覆盖日志。如果多次重新启动 / 继续运行，workflow_run id 可用于从特定运行中选择信息。

### State：
pyFlow 通过在文件中标记其状态来继续作业，而不是通过查找文件目标的存在来继续作业。这是与 make 的主要区别，在重新启动中断的工作流时必须牢记。

每个任务的 runstate 都在 pyflow.data/state/pyflow_tasks_runstate.txt，每个任务的描述都在 pyflow.data/state/pyflow_tasks_info.txt。在每次运行开始时，任何现有的任务文件都在 pyflow.data/state/backup 中备份。

### 其他：

#### 电子邮件通知：

当运行带有 mailTo 参数中给出的一个或多个电子邮件地址的工作流时，pyflow 将尝试在任何不存在主机硬件故障的情况下发送描述运行结果的通知。该电子邮件应来自 3 个结果中的 1 个：（1）成功运行完成（2）第一个不可恢复的任务失败，并描述错误（3）未处理的软件异常。邮件默认来自“pyflow-bot @ YOURDOMAIN”（可配置）。请注意：（1）您可能必须将自动检测到的域中的电子邮件地址更改为接收电子邮件，以及（2）您可能需要检查垃圾邮件过滤器以接收通知。最好配置一个演示脚本，以便在开始生产运行之前通过电子邮件向您发送新机器以测试任何问题。

#### Graph 输出：

pyFlow 提供了一个脚本，可用于生成当前任务依赖关系的图形，其中每个节点按任务状态着色。在 pyflow 状态目录中为每次运行自动创建图生成脚本：

```
pyflow.data/state/make_pyflow_task_graph.py
```
可以不带参数运行此脚本，以基于 pyflow.data/state/ 目录中的数据文件以 dot 格式生成当前任务图。

#### 网站配置：
该文件 ${pyflowDir}/src/pyflowConfig.py 包含任何可能需要在新站点配置的 pyflow 变量或函数。这目前包括：

来自：pyflow 的电子邮件地址
每个任务的默认内存
localhost 模式下每个线程可用的默认内存
响应资源请求而给出的 qsub 参数。
