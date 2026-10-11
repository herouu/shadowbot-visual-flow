# 原型字段查询 + 作用域铁律（通用·实测）

写积木前先读本文。核心两句：**字段名一律查权威表，不要凭记忆**；**跨作用域只能靠外层可变容器**。

## 1. 查原型与字段的权威方法
- 权威表：`<影刀安装目录>\Resources\Copilot\knowledge\visual-blocks\block_detail.md`
  - 每行一个原型：`<proto.name>{"description":…,"notes":…,"inputs":[…],"outputs":[…]}`，`inputs` 里带 `type` / `required` / `choices`（`choices` 就是取值白名单，如 `read_way`、`write_range`、`encoding`）。
  - `notes` 字段常写明**必填联动**（例如「launch_way=open 时必填 open_filename，不能用另一个字段代替」）——这类信息比字段名本身更容易踩。
- 查已插入的块：`flow.blocks.list` 返回每块的 `fields`，可用来核对「引擎实际认的字段名/当前值」。
- 现象：fill 报字段无效或整块被丢弃。原因：字段名/取值凭记忆写。做法：先在 block_detail.md 里搜原型名，逐字段比对 `choices`。
- 不确定时的兜底：`wiki.search` / `wiki.read` 查同名条目；仍不确定就先把该原型**单块插进去**测一遍，再批量写。

## 2. 隐式输出与编号错位排查
- 同族**产出**积木的第 N 个实例名字带 N 后缀（第 1 个无后缀），编号是**流程级计数器**，跨不同实例对象仍连续（实测 `excel_data` / `excel_data2` / `excel_data3`…）。
- 现象：中间插入一个同类积木后，运行期取到错误的表，或直接报变量未定义。原因：其后所有 `*N` 引用整体后移一位。
- 排查手法：按**插入顺序**数出同类积木（如所有读表块），逐个核对引用后缀 == 它的序号；不一致就是错位。改完**全流程复扫**一遍，别只看改动处。
- 缓解写法：插入位置尽量追加在末尾；确需中间插入时，同时把后续引用整体 +1。

## 3. 作用域铁律
1. **`try` / `for` / `if` 内产出的变量不外泄**。
   - 现象：块内算好的值，块外读不到/未定义。做法：跨作用域传值一律用**外层（顶层）声明的可变容器**（dict / list），块内**原地 mutate**（`d['k']=v` / `lst.append(v)`），不要「重新赋值一个新变量」。
2. **产出实例/句柄的积木必须放 `try` 之前（顶层）**。
   - 现象：放进 `try` 内后，try 外的正常出口收尾和 `catch` 都拿不到该句柄。报错形态二选一：`variable_out_of_scope`，或**文案误导**的 `output_id_used_as_variable_name`（看着像输出名写错，**实为作用域问题，不是命名写错**——别去改名）。
   - 正确块序骨架：
```js
[ {prototype_name:'<产出句柄的积木>', inputs:{/* ... */}, outputs:{handle:'h1'}}, // 顶层，先产出
  {prototype_name:'programing.try'},
  {prototype_name:'<读/处理/写>'},
  {prototype_name:'programing.catch'},                                          // 打异常 + 收尾
  {prototype_name:'programing.endtry'},
  {prototype_name:'<顶层收尾>'} ]
```
3. 取舍与兜底：顶层产出的**第 1 个**句柄不会进 `catch` 收尾（作用域规则决定的取舍，不是 bug）。做法：用**状态字典**记录「哪些资源还开着」——产出时置位、正常收尾后清位，`catch` 里遍历未清位的逐个收尾。

## 4. 异常与可见性
- `programing.catch` 的输出是 `exception`(str)。**必须用 `programing.log` 打印 `str(exception)`**。
  - 现象：块内任何异常都被吞成一句固定文案，只能靠猜。原因：catch 没打异常。做法：catch 第一块就是 `programing.log(type:'error', text: str(exception))`。
- 「尽力收尾后继续往上抛」用 `programing.raise`（`reason` 是 str，可传表达式）。
- **增益型环节（非核心链路）建议 best-effort**：写日志/告警即可，不要 raise，避免一个可选步骤失败就阻断主链。

## 5. 类型严格性（数值参数）
- 现象：报「参数start_number类型错误」，且该异常**击穿整段 try**，日志外显成与本环节无关的文案。原因：`start_number`/`stop_number` 是 `int`，传了 float（例如含 `/` 的表达式结果）。
- 做法：参数化延时用显式 int 转换：
```js
{ value_type:'int', value: {expression:'int(max(1.0/qps, 3.0))'} }
```
- 通用检查：凡 `type` 为 `int` 的字段，表达式结果必须显式 `int(...)`；`float` 字段则别传字符串。

## 6. 表达式沙箱（`programing.variable` 的 expression）
- 只自动 import `xbot` / `xbot_visual` / `time` / `glv`（全局变量用 `glv['name']` 引用），并覆写了 `print`。
- 其它 stdlib 一律内联 `__import__('mod')`，例如 `__import__('json').dumps({...})`、`__import__('re').match(...)`；写裸名 `re.match(...)` 会报「变量 xxx 未被定义」（**这个文案会误导成变量问题，其实是缺 import**）。
- 适合在**单条表达式**里做聚合（列表推导 / lambda），比循环积木更可靠；多语句或真 `import` 行请改用代码段类积木。
- 结果要落到顶层容器才能在 try 外读（见 §3）。

## 7. 结构配对自检（离线，不依赖影刀）
- 现象：建流报结构错/流程错位，但单块看起来都对。原因：开启块与收尾块数量/顺序不匹配。
- 做法：建流前对自己的 blocks 数组做**栈式配对**（纯文本检查，快且能定位到块序号）：
```python
CLOSER = {'workflow.endloop': {'workflow.forin','workflow.for','workflow.while',
                               'workflow.infinite_loop','workflow.loop_dict'},
          'workflow.endif':   {'workflow.if','workflow.multiple_conditions_if'},
          'programing.endtry':{'programing.try'}}
LOOP_OPEN = CLOSER['workflow.endloop']
MID = {'workflow.else','workflow.elseif','workflow.multiple_conditions_elseif'}

def check(names):                      # names = [b['prototype_name'] for b in blocks]
    stack = []
    for i, n in enumerate(names):
        if n in ('workflow.break', 'workflow.continue'):
            assert any(s in LOOP_OPEN for s, _ in stack), f'第{i}块 {n} 不在循环内'
        elif n in CLOSER:
            assert stack and stack[-1][0] in CLOSER[n], \
                f'第{i}块 {n} 配对失败，栈顶={stack[-1] if stack else None}'
            stack.pop()
        elif n in MID:
            assert stack and stack[-1][0] in CLOSER['workflow.endif'], f'第{i}块 {n} 不在 if 内'
        elif n in LOOP_OPEN | CLOSER['workflow.endif'] | {'programing.try'}:
            stack.append((n, i))
    assert not stack, f'未闭合: {stack}'
```
- 顺带能查出的另一类错：把「判断/循环」的收尾块写在其开启块之前（栈顶不匹配会立刻报出来）。

## 8. 调用子流程
- `process.run` 的 `process` 字段填**流程 id / 文件名**，**填显示名不行**。
- 入口流程不能被调用（调用入口会直接报错）。
- 父流程**构建期就要解析子流程 id** → 正确顺序是**先建子流程、后建父流程**；`build all` 之类的批处理必须按依赖排序。
- 现象：父流程 fill 报找不到目标流程。原因：子流程还没建/名字改动。做法：构建脚本里把子流程名 → flow_id 的映射显式维护，建父之前先断言映射已解析到。
