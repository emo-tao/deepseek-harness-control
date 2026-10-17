# Harness 的 Sky 操作与故障恢复

这是 `computer-use:computer-use` 的补充，不替代当前运行时的 API 文档。运行时升级后，以实际返回对象与受支持 API 为准；不要复制旧窗口 ID、索引或屏幕坐标。

## 初始化与唯一窗口

在 `node_repl` 执行一次初始化，再在独立调用里枚举实际应用：

```javascript
if (!globalThis.sky) {
  const { sky } = await import('@oai/sky');
  globalThis.sky = sky;
}
```

```javascript
globalThis.harnessApps = await sky.list_apps();
nodeRepl.write(JSON.stringify(
  harnessApps.filter(a => /deepseek|harness/i.test(
    (a.displayName || '') + ' ' + a.id)), null, 2));
```

检查返回项后，以用户指定会话和观察到的应用对象筛选，要求恰好一个窗口。不要把筛选出的第一项当唯一项；没有匹配时列出有限候选并处理歧义。恢复已知运行窗口可用 `list_windows()`。

将唯一的**返回对象**保存在 `harnessReturnedWindow` 后：

```javascript
globalThis.harnessWindow = await sky.get_window({
  id: harnessReturnedWindow.id,
  app: harnessReturnedWindow.app,
});
globalThis.harnessState = await sky.get_window_state({
  window: harnessWindow,
});
```

停下来检查截图是否真是 Harness 及目标会话。状态捕获自动显示的截图直接检查，不为再次查看而解码、保存或重复输出 payload。

## 一次观察、一次操作、立即刷新

选择当前截图/树中的控件后，在下一调用执行一个操作并刷新。坐标只能来自当前截图且已确认映射；索引只能来自当前可访问性树。不要把示例中的变量替换成上一次会话的缓存值。

焦点验证需要 `include_text: true`；只读进度检查通常只需截图或一段有界文本。不要把 `document_text` 整段输出，它可能包含整个长对话；只提取最新公开回复和状态，不采集隐藏推理或凭据。

输入流程：观察编辑框 → 点击 → 刷新确认焦点 → 下一调用 `type_text` → 刷新检查正文 → 依据该次观察点发送 → 刷新检查新消息与执行状态。发生输入异常后，重新观察再决定是否操作。

## 常见问题与有界恢复

| 实际现象 | 恢复方式 |
|---|---|
| 返回窗口叫 Harness，但截图是 Codex/其他应用 | 不输入；重新列窗、唯一选定并 `get_window`；必要时激活正确窗口，再重取状态 |
| 最小化窗口恢复后暂时白屏 | 用返回的窗口对象激活并重取；允许短暂等待后再观察一次，不在白屏中点坐标 |
| 截图被裁剪或有非零 origin | 不猜坐标换算。输入 API 的 x/y 是窗口坐标；截图 origin 是屏幕坐标，两者不足以推导窗口原点。优先可访问性控件，或恢复可确认映射的完整窗口后重取状态 |
| 为恢复完整窗口需最大化 | 激活正确窗口，打开其系统菜单并观察，再选择实际显示的“最大化”控件/快捷键；布局改变后重新截图。不要盲用固定语言快捷键 |
| `set_value` 返回成功但正文没变化 | 重新观察，确认编辑面与原有草稿；正确聚焦后用 `type_text`，核实文字真正出现 |
| 发送调用超时/被用户输入打断 | 接收状态未知；先查看新消息与运行状态，不能盲重发或把停止按钮当发送按钮 |
| 模态框出现、用户切换会话/窗口 | 旧观察失效。监控对象保持原指定会话；恢复该对象前不输入 |
| 锁屏、权限/认证界面、目标无法唯一识别 | 停止输入并报告具体阻塞；不要尝试绕过 |
| helper 轻量调用超时或句柄过期 | 依当前 Computer Use 恢复规则作有界重试/重新绑定；不能自行无限循环或启动替代控制服务 |

每次 UI 输入只用该环境允许的 Computer Use API。不要通过 UI 控制终端、运行框、认证或安全设置，也不要用这套流程自动操作 Codex 本身。终端命令如属于任务范围，应走专用执行工具，并单独遵守相应规则。

## 验证时保留哪些证据

通常记录：目标会话、观察时间、当前阶段、最近的具体进展/错误、任务提交是否确认、约定输出目录及其核验结果。无需保存完整截图历史和全量会话。共享仓库不得包含真实截图、私人路径、会话转储、访问令牌或自动化任务的用户数据。
