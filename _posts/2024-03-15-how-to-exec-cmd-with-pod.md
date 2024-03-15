---
toc: true
title: "如何使用api在pod中执行命令?"
categories:
  - devops
tags:
  - k8s
---

# 痛点

    研发经常来找，生产环境java进程使用了多少内存？与某个域名网络通不通啊？帮我执行下jmap? 帮我执行下jstack ? ...

# 分析

    使用k8s API 实现一种类似web-terminal的效果， 允许研发执行 白名单内的命令 进而在pod内进行一定范围的debug.

# 关键字

- connect_get_namespaced_pod_exec
- xterm.js

# 方案

后端: wesocket + django

前端: element-plus + xterm.js

# 伪代码

后端使用channel 实现 websocket https://channels.readthedocs.io/en/latest/

k8s.py
```python
    def pod_terminal(self, pod_name, ns, container, cmd):
        return stream(
            self.v1.connect_get_namespaced_pod_exec,
            pod_name,
            ns,
            container=container, # pod 中可能存在多个容器, 需要指定具体登录哪一个
            command=cmd,
            stdin=True,
            stdout=True,
            stderr=True,
            tty=False, # 不使用tty
            _preload_content=False,
        )
```
什么是tty? https://linux.cn/article-14093-1.html

consumer.py
```python
from channels.generic.websocket import WebsocketConsumer


Authorization = "Authorization: Bearer "


# 异步获取stream 中的输出, 并通过ws 发送前端
class K8STerminalStreamThread(threading.Thread):
    def __init__(self, websocket, container_stream):
        super(K8STerminalStreamThread, self).__init__()
        self.websocket = websocket
        self.stream = container_stream

    def run(self):
        while self.stream.is_open():
            self.stream.update(timeout=1)
            if self.stream.peek_stdout():
                self.websocket.send(
                    text_data=json.dumps({"stdout": self.stream.read_stdout()})
                )
            if self.stream.peek_stderr():
                self.websocket.send(
                    text_data=json.dumps({"stderr": self.stream.read_stderr()})
                )
        self.stream.close()

class K8sPodTerminalConsumer(WebsocketConsumer):
    def connect(self):
        self.stream = None
        self.thread = None
        self.deployment = None 
        self.pod = None
        self.user = None
        self.accept()
        self.whitelist = [] # 白名单, 可从数据从中动态加载

    def _auth(self, msg):
        # 用户认证, exec 这种危险操作, 必须加 认证和审计
        # wesocket 通用的鉴权方式: 建立连接后, 第一条消息发送用户token
        token = msg.replace(Authorization, "").strip()
        try:
            # 使用token 做鉴权...
            self.user = TokenUser(token)
            self._stream()
        except Exception:  # noqa
            self.user = AnonymousUser()

    def _stream(self):
        # 从url 中获取deployment, pod 参数
        self.deployment = self.scope["url_route"]["kwargs"]["deployment"]
        self.pod = self.scope["url_route"]["kwargs"]["pod"]
        logger.debug(f"获取{self.pod}终端流")
        try:
            # 禁止以root身份登录
            cmd = [
                "/bin/sh",
                "-c",
                "TERM=xterm; export TERM; "
                'if [ $(id -u) -eq 0 ]; then echo "deny root user" >&2; else exec /bin/bash || exec /bin/sh; fi;',
            ]
            self.stream = K8s(app.k8s_cfg).pod_terminal(
                self.pod, ns, app.name, cmd
            )
        except Exception as e:
            logger.error(str(e))
            self.close()
            return
        # stream 建立成功后, 启动一个线程获取stream 输出
        self.thread = K8STerminalStreamThread(self, self.stream)
        self.thread.start()

    def receive(self, text_data=None, bytes_data=None):
        # 第一条消息为认证内容
        if text_data.startswith(Authorization):
            self._auth(text_data)
            if self.user.is_authenticated:
                self._stream()
                self.send(text_data=json.dumps({"stdout": "welcom!"}))
            return
        # 判断用户是否已认证
        if self.user is None or not self.user.is_authenticated:
            self.send(text_data=json.dumps({"stderr": "Please login first"}))
            self.disconnect(1)
            return
        # 是否为空
        if not text_data:
            self.send(text_data=json.dumps({"stdout": "\n"}))
            return
        # 处理用户退出
        if text_data == "exit":
            self.send(text_data=json.dumps({"stdout": "bye!"}))
            self.disconnect(1)
            return
        
        # 白名单过滤
        allow = False
        # 判断用户是否为超级管理员
        if self.user.is_superuser():
            allow = True
        else:
            for f in self.whitelist:
                if f.allow(text_data):
                    allow = True
                    break
        if allow:
            # shlex 防注入
            text_data = shlex.join(shlex.split(text_data))
            self.stream.write_stdin(text_data + "\n")
						# 添加审计日志
        else:
            self.send(
                text_data=json.dumps(
                    {
                        "stderr": f"Command `{text_data}` is not allowed, please contact the administrator"
                    }
                )
            )

    def disconnect(self, close_code):
        if self.stream:
            self.stream.write_stdin("exit\n")
            self.stream.close()
        self.close()

```

前端:
```vue
<script setup>
import {Terminal} from 'xterm';
import {FitAddon} from 'xterm-addon-fit'
import 'xterm/css/xterm.css';
import {getToken} from "@/utils/auth";
import {getWhiteListCmd} from "@/api/k8s";
import {useUserStore} from "@/store/user";

const route = useRoute()
const terminalContainer = ref(null);
const userStore = useUserStore()
const terminal = new Terminal(
  {
    rendererType: "canvas", //渲染类型
    disableStdin: false, //是否应禁用输入
    cursorBlink: true, //光标闪烁
    fontSize: 18, //字体大小
    convertEol: true, //是否应将光标移
    cursorStyle: 'underline', //光标样式
    theme: {
      foreground: "#ECECEC", //字体
      background: "#000000", //背景色
      cursor: "help", //设置光标
      lineHeight: 20
    }
  }
);
const socket = ref(null);
const inputCommand = ref('')
const state = reactive({
  whitelistCommand: []
})

// 定义打印$并换行的快捷方式
const prompt = () => {
  terminal.write("\r\n\x1b[33m$\x1b[0m ")
}
// 定义换行的快捷方式
const emptyLine = () => {
  terminal.write("\r\n")
}
// 打印标准输入的快捷方式
const stdout = (data) => {
  // 替换所有的 \n 为 \r\n
  const formattedData = data.replace(/\n/g, "\r\n");
  terminal.write("\x1b[33m" + formattedData + "\x1b[0m ")
  prompt()
}
// 打印标准错误的快捷方式
const stderr = (data) => {
  const formattedData = data.replace(/\n/g, "\r\n");
  terminal.write("\x1b[31m" + formattedData + "\x1b[0m")
  prompt()
}
// 初始化terminal
const initTerminal = () => {
  if (terminalContainer.value) {
    terminal.open(terminalContainer.value);
  }
  // canvas背景全屏
  const fitAddon = new FitAddon()
  terminal.loadAddon(fitAddon)
  fitAddon.fit()
  
  // onKey 处理用户输入, 将用户输入的每个有效字符拼接到 inputCommand
  // 输入回车时发送 inputCommand 
  // 输入删除时 存在bug
  // 此处逻辑可自行修改
  terminal.onKey(e => {
    const printable = !e.domEvent.altKey && !e.domEvent.altGraphKey && !e.domEvent.ctrlKey && !e.domEvent.metaKey
    if (e.domEvent.keyCode === 13) {
      socket.value.send(inputCommand.value)
      inputCommand.value = ''
      emptyLine()
    } else if (e.domEvent.keyCode === 8) { // back 删除的情况
      if (terminal._core.buffer.x > 2) {
        terminal.write('\b \b')
        inputCommand.value = inputCommand.value.slice(0, -1)
      }
    } else if (printable) {
      terminal.write(e.key)
      inputCommand.value += e.key
    }
  });
  // 监听用户输入
  terminal.onData(key => {  // 粘贴的情况
    if (key.length > 1) {
      terminal.write(key)
      inputCommand.value += key
    }
  })
}

// 初始化websocket
const initWS = () => {
  const ws = new WebSocket("ws://xxx");
  ws.onopen = () => {
    // 与后端约定好, 第一条消息为用户token
    ws.send("Authorization: Bearer " + getToken())
  }
  ws.onmessage = (event) => {
    // 接受消息的处理函数, 可自定义
    const data = event.data
    const jd = JSON.parse(data)
    if (jd?.stdout) {
      stdout(jd.stdout)
    }
    if (jd?.stderr) {
      stderr(jd.stderr)
    }

  }
  ws.onclose = () => {
    // 关闭时的处理函数
    ws.send("exit")
  }
  socket.value = ws
}

onMounted(() => {
  const {id, podId} = route.params || {}
  if (id && podId) {
    initTerminal()
    initWS()
  }
});

onBeforeMount(() => {
  if (socket.value) {
    socket.value.send("exit")
    socket.value.close()
  }
})
</script>

<template>
  <div>
    <div class="fd-table_header">
      <strong>Pod控制台</strong>
    </div>
    <div ref="terminalContainer" class="terminal"></div>
  </div>
</template>

<style scoped lang="scss">
.terminal {
  width: 100%;
  height: 100%;
}
</style>

```

