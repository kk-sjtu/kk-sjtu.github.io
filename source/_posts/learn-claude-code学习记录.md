---
title: learn-claude-code学习记录
date: 2026-05-03 17:27:20
tags:
---

# lcc chapter1 loop
 基本的agent_loop循环，和claw0的一致。有启发的一点是，如果让agent生成一个文件，那么在这个过程中，如何处理context，处理接下来对话的message信息，以节省token。或许是很重要的。

# lcc chapter2 tool-use
* 在这一节中，使用了安全路径来加强了工具调用时的安全性。
* 使用了分发映射表，dispatch map来存储tool。同时使用了**的小技巧去进行字典拆解。用以处理agent返回的内容。

# lcc chapter3 todo
通过schema工具表添加了todo的工具，设置相应的字段。用于在agent对话时保证行动不偏移，设置了N次工具调用后便触发todo的功能，将这条消息作为result传给下一轮对话中。于是下一轮对话时agent便会返回一个带有tool：todo的消息，以及对应的字段的内容。于是将要输入todo工具的字段进行提取，作为传参传入到todomanager的实例化对象中。如果会调用多轮todo，则将前期的记录，记录在todomanager的属性字段中。

# lcc chapter4 subagent
Agent根据用户的提问，结合自己的tool-list、系统提示词，判断是否需要开subagent。subagent的结果将返回到主agent作为输入（即在主message中仅仅加上subagent的最后一条消息，忽略掉sub-message的其他内容），这样即使subagent的message有几十条，但主agent就只有一开始的请求以及最后的result，即subagent的大部分思考过程没有加入到主agent的message中。subagent通过一个subagent做到了会话之间的隔离，防止主对话的污染。这里的subagent是主线程阻塞的，没有另起后台线程。而claw0中的使用了asynic库的多线程操作，做到了后台调用子agent。可能后续lcc也会加多线程操作。

# lcc chapter5 skill_loading
这一节的核心是渐进式披露，或者是按需披露。系统提示词相当于全局变量。加入skill的目录与简介。于是在agent的每一轮对话中，都会分析是否使用哪种skill。如果确认使用，则去读取对应的skill文件，加入上下文。这里的读取对应skill，看做成了一个工具调用。
此外，写skill也很重要。

# lcc chapter6 Context Compact
三层压缩。通过压缩上下文来节省token。
* 第一层：micro_compact，每次调用llm前，将旧的tool-result替换为占位符。战略性丢失。丢旧的，长的，但是代码读取的文件内容不丢失（因为丢失了会变笨。同时还会逼迫agent再次重读文件）。直接修改message里的内容。
* 第二层：auto_compact，每次上下文超出token阈值时，使用大模型压缩的方法，进行摘要。并对原始消息进行持久化。具体使用方法是，检测到需要自动压缩时，在自动压缩函数中新调用一个max_token较小的llm，重新进行llm会话，并将结果打包为一个summary，然后返回到主线程中，主agent进行调用。同时将原始消息的脚本进行持久化保留。
* 第三层：manual compact，将压缩变为工具。由llm按需触发。
信息没有真正丢失, 只是移出了活跃上下文

# lcc chapter7 task_system

本节添加了一个任务管理系统，有创建任务，更新任务，删除依赖，显示任务的功能。这个任务管理系统，通过tool-use来调用。通过将任务管理系统中的CRUD函数，注入schema中，然后schema作为系统提示词注入llm的输入参数，于是llm可以根据当前提问来选择合适的工具。如果要涉及到任务管理等操作，则去调用这个任务管理系统工具。
不同于第三节的todo，todo是注入到上下文中的内存式的任务管理，而这节的任务管理的内容是持久化在磁盘之上的。

# lcc chapter8 Background Tasks
本节添加了后台任务管理系统，通过工具调用可以实现后台任务启动，即llm可以根据用户提问来判断是否调用工具，调用之后后台任务就会以子进程方式启动。然后在agent_loop内的每次对话循环中，都会先排空一遍消息队列。这个操作是在后台子进程中完成，排空（drain）操作，使用with self._lock，这样可以自动加锁解锁。即运行到排空操作时，如果后台子进程还在_execute,则不会进行排空。直到_execute完成后，解锁，排空操作加锁。然后进行排空。排空其实就是排空子线程的消息队列，并返回给主线程。               
                "task_id": task_id,
                "status": status,
                "command": command[:80],
                "result": (output or "(no output)")[:500],
例如上述格式，即为子线程消息队列的元素格式。一条一条的。只不过，排空时没有考虑‘command’，因为llm不需要知道之前用了什么命令，只需要现在的output即可（还可以减少上下文长度）。此外，’command’信息也可以通过check函数获取。


# lcc chapter9 Agent Teams
这一节，使用了类似sub-agent的方式，但这一节的子agent有生命周期。即通过config.json维护了起来。同时，每一个agent具有它的收件箱，来记录请求。当每个agent执行自己的agent-loop时,调用llm之前，需要将自己的inbox的内容放入到上下文中，同时清空。如果要给其他agent发送消息，则调用send_message工具，然后会给相应的对象发送消息（原子级别的写入一条消息，以json形式）。 
with open(inbox_path, "a") as f:
            f.write(json.dumps(msg) + "\n")
需要明确一点，操作系统读写文件，是天然满足原子性的。所以BUS.read_inbox不需要加锁。

# lcc chapter10 Team Protocols
这一节，在原先的 命令-执行的基础上，增添了更充分的协议。以关机和计划审批为例。如果是第九章，那么lead让subagent关机，它就会关机（或许不会，但应该会吧），而当我们引入一个沟通机制，即通过工具的方式，告诉lead，你如果要求关机时，需要在一个shutdown队列中添加request，同时告诉subagent你可以同意也可以拒绝（即，将shutdown-response写入subagent的loop中的工具Schema中）。于是shutdown-response作为一种工具，于是当lead发出request时，subagent能够根据实际情况决定是否同意或拒绝关机。并将结果，写入shutdown队列。
同理，计划审批则是subagent向lead提出的request。和shutdown的数据流动方向相反。也是添加了类似的审核机制。
线程锁的目的是为了防止lead和subagent同时读写一个共享数据。

# lcc chapter11 Autonomous Agents
给agent的团队增添了自主性。让lead agent自主分配任务。如果用户对话中提到了具体agent，例如让alice干某事，那么lead agent的线程中就会spawn alice的线程，并给他们发送邮件。alice线程启动后，会先读取自己的inbox，然后注入上下文中，并清空。也有可能会回复shutdown-request。之后alice就可以根据inbox中的命令，去进行工作了。
如果用户对话中没有提到具体的agent，则需要lead自主分配。采用轮询teammate的方式，可能会遇到teammate working的情况，则需要跳过。直到找到空闲。如果都是working，则需要自己创建新的agent。这些都是由lead agent自主决定。即自治。

判断空闲与否的方式，是agent去进行config.json表中进行read_file，然后识别出idle字眼后，自主选择IDLE工具。这样，agent的下一个对话的block.name就是idle。如果是这样，就进行一个空闲标记。只有进入空闲态时，sub-agent才可以去看任务看板。

# lcc chapter11 Worktree + Task Isolation
虽然文件写入是原子级的，但是如果多agent共享一个目录则会出现无法回滚的问题。"干净回滚"指的是能够可靠地撤销某个 Agent 做的所有改动，而不影响其他 Agent 的工作。
这里的worktree是独立的目录。即物理复制两份目录，相互独立，然后每个agent在独立目录上修改。这不同于git branch，后者checkout
切换的是同目录下的不同分支。git branch往往是不需要同时运行的单线程任务，然后切换来控制版本。而worktree往往需要，多个目录下的项目进行并行操作。
当用户提出需求后，主agent进行创建任务，并根据要求去创建不同worktree，每一个worktree绑定着某项任务。同时任务.json中也保存着所属的worktree，并修改任务的status。每个worktree都会开启后台线程来进行子任务。这一节没有涉及到subagent，用子线程执行bash命令的方式进行了替代。为了持久化历史记录，在每一次线程操作前后都会emit一个event记录，保存在event.json中。
每个worktree执行完任务后，可以由agent自主选择remove整个目录或者keep。
