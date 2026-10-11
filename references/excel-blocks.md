# Excel 类积木语义（`excel.*`）

适用：任何「读表 → 处理 → 写回」的可视化流程。字段名以影刀原型 schema 为准，实测口径写在下面。

## 1. 打开 / 关闭

- `excel.launch`：`launch_way=create` 必填 `save_filename`，`launch_way=open` 必填 `open_filename`，**两个字段不能互换**（填错的那个被静默忽略，run 时才报路径为空）。
- `driver_way=openpyxl` 时**不驱动 Excel 界面**：宏、透视表、PDF 导出、大部分格式/选择性粘贴都不可用，纯数据读写够用。
- 输出 `excel_instance` 是影刀包装对象，**不是原生 openpyxl Workbook**；Python 片段里用 `excel_instance.workbook`，取表用 `excel_instance.get_sheet_by_name(name).worksheet`。
- **实例名不可自定义**：第 1 个打开块 → `excel_instance`，第 2 个 → `excel_instance2`，依次递增。改输出名会导致整块 fill 被丢弃。
- `excel.close`：`operation` ∈ `close_specified`/`close_all`；`close_way` ∈ `notsave`/`save`/`saveas`。
- **只读用途的工作簿一律 `close_way=notsave`**（人工维护的模板 / 参数表 / 字典表）：否则程序退出时会把人工维护的内容整体重写一遍。
  反例：一张人工排好版、带公式与批注的参数表被 `save` 回写后，列宽与公式丢了，下次打开面目全非。
- 收尾：**无论成败都要关**已打开的工作簿，异常出口也要尽力关；否则下次运行直接撞「文件被占用」。

## 2. 读取

- `read_way=used_range` + `has_header_row=true` **仍返回表头行** → 消费端一律 `data[1:]`。
  反例：把 `has_header_row=true` 当成"自动去掉表头"，结果表头被当成数据行参与排序，多出一条脏记录。
- `used_range` 会夹带**空行 / 短行**（尾部空行、中间缺列）→ 使用前先过滤：**主键非空 且 列数 ≥ 期望列数**。否则 `row[i]` 下标越界整轮抛异常。
- `sheet_name` 缺省时取活动表；多表簿建议显式指定，别依赖"最后打开的那张"。
- 列相关参数（`cell_column_name`/`column_column_name` 等）是 **Excel 列字母或数字**（`A` / `1`），**不是表头文字**：传中文表头能过语法校验，但 `app.run` 时报「已超出最大列 XX」。
- 输出数据变量 `excel_data` / `excel_data2` …：**流程级全局递增**，跨不同工作簿连续编号，不按簿各自计数。见 §5。

## 3. 写入

- `write_range=area` 必填 **`row_num` + `column_name`**；`begin_row_num`/`begin_column_name` 只用于 `row`/`column` 模式，混用会一直报缺字段。实测 `programing.databook.write_data_to_workbook` 同理，且**字段列表不标注哪些是当前模式必填，一次校验只报一个缺失字段** → 提前查清矩阵，别一轮一轮试。
- `write_way=override` **只覆盖写入区，不清尾部**：目标表原有更长的行会**残留**。
  反例：表从 90 行缩到 10 行，只写前 10 行 → 后 80 行旧数据仍在，下游把它们当成有效行。
  修法：写入区按目标长度**补齐空值覆盖**，或显式清空多余行。
- 写入内容里的**每个单元格值都要能被下游安全解析**：空值不要写成字符串 `'None'`；数值列写数值类型。
  反例：一列里混进 `'20万+'` 这种文本，下游 `int(float(x))` 一行脏值就 `ValueError`，整轮作废。

## 4. 写回五条口径（读表 → 处理 → 写回通用）

1. **只改目标行**：只有需要变更的行才改，其余行**整行原样透传**。切勿对全表做强制类型转换 —— 一行脏值就抛异常，整轮结果全丢。
2. **哨兵值**：用 `-1` / 空串表示「未处理 / 未命中」，与**合法的 0 区分开**；取值用 `dictionary.get_value(key, default=-1)` 之类显式默认值。
3. **写盘 gate 用「变更行数 > 0」**，不要用「容器长度 > 0」。
   反例：本轮所有结果都是 0 时，容器非空 → 白改一遍整表，还打出一条"完成"日志，掩盖了实际什么都没变。
4. **防空覆盖**：本轮结果为空 / 命中为 0 → **跳过写入**，只写告警与日志，保住上一轮的好数据。
5. **收尾**：成功路径与异常路径都要 **save + close**；`catch` 里补一次尽力收尾。
   （`launch` 产出实例的积木必须放在 `try` **之前**，否则 catch 拿不到句柄、填块直接报作用域错。）

## 5. 编号位移：改完必须全流程复核

- 序号是**流程级计数器**，不是每个簿/每段各自的：中间插入一个打开或读取块，其后所有 `excel_instanceN` / `excel_dataN` 引用**整体后移**。
- 运行期表现：取到错的表（静默错值）或直接报变量未定义。
- 加/删读块后，**全流程 grep 一遍 `excel_instance`/`excel_data` 引用**再保存，别只看被改的那一段。
