---
title: "Streamlit API速查表"
description: "Streamlit是一个机器学习与数据科学领域的可视化库，可以帮助我们轻松搭建web页面"
date: 2026-05-13
lastmod: 2026-05-13
weight: 4
categories:
    - Tutorial
    - Tech Stack
tags:
    - 教程
    - 技术栈
    - streamlit
---


# Streamlit 官方API速查表（含功能与用法）

- streamlit是一个开源的数据科学工具，可以让我们很方便的创作出精美的UI界面并部署，以下是其API速查表，包含每个API的核心功能和基础用法示例：

---

### Write and magic:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.write` | 通用输出函数，可渲染文本、数据、图表、对象等几乎所有类型内容 | `st.write("Hello Streamlit!")` <br> `st.write(pd.DataFrame({'A': [1,2], 'B': [3,4]}))` |
| `st.write_stream` | 流式输出内容（如逐行打印、实时日志），支持生成器/迭代器 | `def stream():` <br> `    for word in ["Hello", "Streamlit", "!"]:` <br> `        yield word + " "` <br> `        time.sleep(0.5)` <br> `st.write_stream(stream)` |
| `magic` | 魔法命令，无需调用函数直接输出变量/表达式（Streamlit自动识别） | `df = pd.DataFrame({'A': [1,2]})` <br> `df  # 直接写变量名，等价于st.write(df)` <br> `"计算结果：", 1+2  # 直接写表达式` |

---

### Text elements:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.title` | 设置页面主标题（一级标题，最大字号） | `st.title("我的Streamlit应用")` |
| `st.header` | 设置二级标题（次于title） | `st.header("数据可视化模块")` |
| `st.subheader` | 设置三级标题（次于header） | `st.subheader("销售额分析")` |
| `st.markdown` | 渲染Markdown格式文本（支持标题、列表、链接、加粗等） | `st.markdown("### 加粗文本：**Streamlit**")` <br> `st.markdown("- 列表项1\n- 列表项2")` |
| `st.badge` | 生成带样式的徽章（如状态、链接、版本号） | `st.badge("https://streamlit.io", label="官方网站")` <br> `st.badge("v1.32.0", label="版本")` |
| `st.caption` | 渲染小号说明文本（如图片/表格注释、页脚） | `st.caption("数据来源：公司2024年销售报表")` |
| `st.code` | 渲染代码块，支持语法高亮、指定编程语言 | `st.code("import pandas as pd\nprint(df.head())", language="python")` |
| `st.divider` | 渲染水平分隔线，用于内容分区 | `st.divider()` |
| `st.echo` | 执行代码并同时显示代码本身和执行结果（调试/演示） | `with st.echo():` <br> `    x = 10` <br> `    y = 20` <br> `    st.write(x + y)` |
| `st.latex` | 渲染LaTeX数学公式 | `st.latex(r"\sum_{i=1}^n x_i = \bar{x} \times n")` |
| `st.text` | 渲染纯文本（无格式，适合简单文本输出） | `st.text("这是一段无格式的纯文本")` |
| `st.help` | 显示指定对象的帮助文档（如函数、类的说明） | `st.help(st.write)` <br> `st.help(pd.DataFrame)` |
| `st.html` | 渲染原始HTML代码（支持自定义样式/布局） | `st.html("<div style='color:red;'>红色HTML文本</div>")` |
| `st.iframe` | 嵌入外部网页/资源的iframe | `st.iframe("https://www.baidu.com", width=800, height=400)` |

---

### Data elements:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.dataframe` | 交互式展示DataFrame（支持排序、筛选、翻页） | `df = pd.DataFrame({'A': [1,2,3], 'B': [4,5,6]})` <br> `st.dataframe(df, width=600, height=300)` |
| `st.data_editor` | 交互式编辑DataFrame（支持修改单元格、新增/删除行） | `edited_df = st.data_editor(df)` <br> `st.write("修改后的数据：", edited_df)` |
| `st.column_config` | 自定义st.dataframe/st.data_editor的列配置（类型、标签、样式） | `config = {` <br> `    "A": st.column_config.NumberColumn("编号", min_value=0),` <br> `    "B": st.column_config.TextColumn("数值")` <br> `}` <br> `st.dataframe(df, column_config=config)` |
| `st.table` | 静态表格展示（无交互，适合简单数据展示） | `st.table(df)` |
| `st.metric` | 展示关键指标（含标题、数值、同比变化） | `st.metric(label="今日销售额", value="¥10000", delta="+12%")` |
| `st.json` | 格式化展示JSON数据（支持折叠/展开） | `st.json({"name": "Streamlit", "version": "1.32.0", "features": ["easy", "fast"]})` |

---

### Chart elements:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.area_chart` | 绘制面积图（基于DataFrame） | `st.area_chart(df, x="A", y="B")` |
| `st.bar_chart` | 绘制柱状图（快速可视化分类数据） | `st.bar_chart(df, x="A", y="B", color="#ff0000")` |
| `st.line_chart` | 绘制折线图（适合展示时间序列/趋势） | `st.line_chart(df, x="A", y="B")` |
| `st.map` | 绘制交互式地图（基于经纬度数据） | `map_df = pd.DataFrame({` <br> `    "lat": [39.9042, 31.2304],` <br> `    "lon": [116.4074, 121.4737]` <br> `})` <br> `st.map(map_df)` |
| `st.scatter_chart` | 绘制散点图（展示两个变量的相关性） | `st.scatter_chart(df, x="A", y="B", size=100)` |
| `st.altair_chart` | 渲染Altair交互式图表（自定义可视化） | `import altair as alt` <br> `chart = alt.Chart(df).mark_point().encode(x="A", y="B")` <br> `st.altair_chart(chart, use_container_width=True)` |
| `st.graphviz_chart` | 渲染Graphviz图形（如流程图、思维导图） | `st.graphviz_chart("digraph {A -> B; B -> C;}")` |
| `st.plotly_chart` | 渲染Plotly交互式图表（支持复杂可视化） | `import plotly.express as px` <br> `fig = px.scatter(df, x="A", y="B")` <br> `st.plotly_chart(fig, use_container_width=True)` |
| `st.pydeck_chart` | 绘制3D地图/地理空间可视化 | `import pydeck as pdk` <br> `layer = pdk.Layer(` <br> `    "ScatterplotLayer", data=map_df, get_position=["lon", "lat"], radius=100000` <br> `)` <br> `st.pydeck_chart(pdk.Deck(layers=[layer]))` |
| `st.pyplot` | 渲染Matplotlib图表 | `import matplotlib.pyplot as plt` <br> `plt.plot(df["A"], df["B"])` <br> `st.pyplot(plt)` |
| `st.vega_lite_chart` | 渲染Vega-Lite图表（轻量级交互式可视化） | `spec = {` <br> `    "mark": "bar", "encoding": {"x": {"field": "A"}, "y": {"field": "B"}}` <br> `}` <br> `st.vega_lite_chart(df, spec=spec)` |

---

### Input widgets:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.button` | 生成点击按钮（触发回调/逻辑） | `if st.button("点击我"):` <br> `    st.write("按钮被点击了！")` |
| `st.download_button` | 生成下载按钮（下载文件/数据） | `csv = df.to_csv(index=False)` <br> `st.download_button(` <br> `    label="下载CSV", data=csv, file_name="data.csv", mime="text/csv"` <br> `)` |
| `st.form_submit_button` | 表单提交按钮（需配合st.form使用） | `with st.form("my_form"):` <br> `    name = st.text_input("姓名")` <br> `    submit = st.form_submit_button("提交")` <br> `if submit:` <br> `    st.write("你好，", name)` |
| `st.link_button` | 生成跳转链接按钮 | `st.link_button("前往Streamlit官网", "https://streamlit.io")` |
| `st.menu_button` | 生成下拉菜单按钮（包含多个选项） | `menu = st.menu_button("操作菜单", options=["新增", "编辑", "删除"])` <br> `if menu == "新增":` <br> `    st.write("执行新增操作")` |
| `st.page_link` | 生成跨页面跳转链接（多页面应用） | `st.page_link("pages/analysis.py", label="数据分析页面", icon="📊")` |
| `st.checkbox` | 生成复选框（单/多选项） | `agree = st.checkbox("我同意条款")` <br> `if agree:` <br> `    st.write("已同意")` |
| `st.color_picker` | 生成颜色选择器（获取RGB/十六进制颜色） | `color = st.color_picker("选择颜色", "#ff0000")` <br> `st.write("选中的颜色：", color)` |
| `st.feedback` | 生成用户反馈组件（星级/拇指评价） | `feedback = st.feedback("stars")  # 或 "thumbs"` <br> `if feedback:` <br> `    st.write("你的评分：", feedback)` |
| `st.multiselect` | 生成多选下拉框（支持选择多个选项） | `options = st.multiselect("选择水果", ["苹果", "香蕉", "橙子"], default=["苹果"])` <br> `st.write("选中：", options)` |
| `st.pills` | 生成胶囊样式的选择器（视觉友好的多选/单选） | `pill = st.pills("选择类别", ["A类", "B类", "C类"], index=0)` <br> `st.write("选中：", pill)` |
| `st.radio` | 生成单选按钮组 | `choice = st.radio("选择性别", ["男", "女", "其他"])` <br> `st.write("你的选择：", choice)` |
| `st.segmented_control` | 生成分段控制器（移动端风格的单选） | `seg = st.segmented_control("选择视图", ["列表", "网格", "详情"])` <br> `st.write("当前视图：", seg)` |
| `st.selectbox` | 生成单选下拉框 | `option = st.selectbox("选择城市", ["北京", "上海", "广州"])` <br> `st.write("选中：", option)` |
| `st.select_slider` | 生成选择滑块（离散值选择） | `slider = st.select_slider("选择评分", options=[1,2,3,4,5], value=3)` <br> `st.write("评分：", slider)` |
| `st.toggle` | 生成开关按钮（布尔值选择，替代checkbox） | `toggle = st.toggle("开启功能")` <br> `if toggle:` <br> `    st.write("功能已开启")` |
| `st.number_input` | 生成数字输入框（整数/浮点数，支持范围限制） | `num = st.number_input("输入数字", min_value=0, max_value=100, value=50)` <br> `st.write("输入的数字：", num)` |
| `st.slider` | 生成滑动条（连续/离散数值选择） | `slider = st.slider("选择温度", min_value=0, max_value=100, value=25)` <br> `st.write("温度：", slider)` |
| `st.date_input` | 生成日期选择器 | `date = st.date_input("选择日期", value=datetime.date.today())` <br> `st.write("选中的日期：", date)` |
| `st.datetime_input` | 生成日期+时间选择器 | `dt = st.datetime_input("选择时间", value=datetime.datetime.now())` <br> `st.write("选中的时间：", dt)` |
| `st.time_input` | 生成时间选择器 | `time = st.time_input("选择时间", value=datetime.time(12, 0))` <br> `st.write("选中的时间：", time)` |
| `st.chat_input` | 生成聊天输入框（适配聊天机器人场景） | `prompt = st.chat_input("请输入消息...")` <br> `if prompt:` <br> `    st.write("你说：", prompt)` |
| `st.text_area` | 生成多行文本输入框 | `text = st.text_area("输入备注", value="默认文本", height=100)` <br> `st.write("输入的文本：", text)` |
| `st.text_input` | 生成单行文本输入框 | `name = st.text_input("输入姓名", placeholder="请输入你的姓名")` <br> `if name:` <br> `    st.write("你好，", name)` |
| `st.audio_input` | 生成音频录制/上传组件 | `audio = st.audio_input("录制/上传音频")` <br> `if audio:` <br> `    st.audio(audio)` |
| `st.camera_input` | 生成摄像头拍照组件 | `img = st.camera_input("拍照")` <br> `if img:` <br> `    st.image(img)` |
| `st.file_uploader` | 生成文件上传组件（支持多文件、指定类型） | `file = st.file_uploader("上传文件", type=["csv", "xlsx"])` <br> `if file:` <br> `    df = pd.read_csv(file)` <br> `    st.dataframe(df)` |

---

### Media elements:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.audio` | 播放音频文件（支持mp3/wav等格式） | `st.audio("audio.mp3", format="audio/mp3", start_time=10)` <br> `# 或播放二进制音频` <br> `st.audio(audio_bytes, format="audio/wav")` |
| `st.image` | 展示图片（支持本地/网络图片，多图） | `st.image("image.png", caption="示例图片", width=300)` <br> `# 多图展示` <br> `st.image(["img1.png", "img2.jpg"], caption=["图1", "图2"])` |
| `st.logo` | 展示应用Logo（固定在侧边栏/顶部，视觉优先级高） | `st.logo("logo.png", icon_image="favicon.png")` |
| `st.pdf` | 展示PDF文件（支持翻页、缩放） | `st.pdf("document.pdf", width=800, height=600)` |
| `st.video` | 播放视频文件（支持本地/网络视频） | `st.video("video.mp4", format="video/mp4", start_time=5)` <br> `# 网络视频` <br> `st.video("https://www.youtube.com/watch?v=xxx")` |

---

### Layouts and containers:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.columns` | 生成多列布局（横向分栏） | `col1, col2 = st.columns(2)` <br> `with col1:` <br> `    st.write("第一列内容")` <br> `with col2:` <br> `    st.write("第二列内容")` |
| `st.container` | 生成通用容器（用于分组内容、动态更新） | `container = st.container()` <br> `container.write("容器内内容")` <br> `# 后续更新容器` <br> `container.write("新增内容")` |
| `st.dialog` | 生成弹窗（模态框，需手动触发打开） | `dialog = st.dialog("弹窗标题")` <br> `if st.button("打开弹窗"):` <br> `    with dialog:` <br> `        st.write("弹窗内内容")` <br> `        st.button("关闭", on_click=dialog.close)` |
| `st.empty` | 生成空容器（用于占位、动态替换内容） | `placeholder = st.empty()` <br> `placeholder.write("初始内容")` <br> `# 替换内容` <br> `placeholder.write("替换后的内容")` |
| `st.expander` | 生成可折叠/展开的面板（节省空间） | `with st.expander("查看更多内容"):` <br> `    st.write("折叠面板内的详细内容")` |
| `st.form` | 生成表单容器（批量收集输入，统一提交） | `with st.form("my_form"):` <br> `    name = st.text_input("姓名")` <br> `    age = st.number_input("年龄")` <br> `    submit = st.form_submit_button("提交")` <br> `if submit:` <br> `    st.write(f"姓名：{name}，年龄：{age}")` |
| `st.popover` | 生成悬浮弹窗（点击按钮展开，非模态） | `with st.popover("悬浮弹窗"):` <br> `    st.write("悬浮弹窗内的内容")` <br> `    st.button("确认")` |
| `st.sidebar` | 生成侧边栏（放置导航/筛选组件） | `with st.sidebar:` <br> `    st.slider("侧边栏滑块", 0, 100)` <br> `    st.selectbox("侧边栏选择框", ["A", "B"])` |
| `st.space` | 生成空白空间（调整组件间距） | `st.write("内容1")` <br> `st.space(height=50)  # 50px空白` <br> `st.write("内容2")` |
| `st.tabs` | 生成标签页（多内容页切换） | `tab1, tab2 = st.tabs(["标签1", "标签2"])` <br> `with tab1:` <br> `    st.write("标签1内容")` <br> `with tab2:` <br> `    st.write("标签2内容")` |

---

### Chat elements:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.chat_input` | 聊天输入框（同Input widgets，适配聊天场景） | 见Input widgets中`st.chat_input` |
| `st.chat_message` | 生成聊天消息气泡（区分用户/助手） | `with st.chat_message("user"):` <br> `    st.write("你好！")` <br> `with st.chat_message("assistant"):` <br> `    st.write("您好，有什么可以帮助的？")` |
| `st.status` | 生成状态面板（展示操作进度/结果，可折叠） | `with st.status("处理中...", expanded=True) as status:` <br> `    st.write("步骤1：加载数据")` <br> `    time.sleep(1)` <br> `    st.write("步骤2：处理数据")` <br> `    status.update(label="处理完成", state="complete", expanded=False)` |
| `st.write_stream` | 流式输出聊天回复（同Write and magic） | 见Write and magic中`st.write_stream` |

---

### Status elements:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.success` | 展示成功提示框（绿色背景） | `st.success("数据加载成功！")` |
| `st.info` | 展示信息提示框（蓝色背景） | `st.info("这是一条提示信息")` |
| `st.warning` | 展示警告提示框（黄色背景） | `st.warning("数据格式异常，请检查！")` |
| `st.error` | 展示错误提示框（红色背景） | `st.error("数据加载失败！")` |
| `st.exception` | 展示异常堆栈信息（调试用） | `try:` <br> `    1 / 0` <br> `except ZeroDivisionError as e:` <br> `    st.exception(e)` |
| `st.progress` | 展示进度条（0-100的数值） | `progress_bar = st.progress(0)` <br> `for i in range(100):` <br> `    progress_bar.progress(i + 1)` <br> `    time.sleep(0.05)` |
| `st.spinner` | 展示加载中动画（上下文管理器） | `with st.spinner("加载数据中..."):` <br> `    time.sleep(3)` <br> `st.success("加载完成！")` |
| `st.status` | 状态面板（同Chat elements，通用状态展示） | 见Chat elements中`st.status` |
| `st.toast` | 展示轻量级提示（右下角弹出，自动消失） | `st.toast("操作成功！", icon="✅")` |
| `st.balloons` | 展示气球动画（庆祝效果） | `if st.button("庆祝"):` <br> `    st.balloons()` |
| `st.snow` | 展示雪花动画（节日效果） | `if st.button("下雪"):` <br> `    st.snow()` |

---

### Authentication and user info:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.login` | 生成登录组件（对接身份验证服务） | `login = st.login("请登录", authentication_type="oauth")` <br> `if login:` <br> `    st.write("登录成功！")` |
| `st.logout` | 生成登出按钮（清除登录状态） | `if st.button("登出"):` <br> `    st.logout()` <br> `    st.rerun()` |
| `st.user` | 获取当前登录用户信息 | `user = st.user` <br> `if user:` <br> `    st.write(f"当前用户：{user.name} ({user.email})")` |

---

### Navigation and pages:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.navigation` | 生成导航菜单（多页面应用的统一导航） | `pages = [` <br> `    st.Page("home.py", title="首页", icon="🏠"),` <br> `    st.Page("analysis.py", title="分析", icon="📊")` <br> `]` <br> `st.navigation(pages)` |
| `st.Page` | 定义页面对象（配合st.navigation） | 见`st.navigation`示例 |
| `st.page_link` | 页面跳转链接（同Input widgets） | 见Input widgets中`st.page_link` |
| `st.switch_page` | 强制切换到指定页面（编程式跳转） | `if st.button("跳转到分析页"):` <br> `    st.switch_page("pages/analysis.py")` |

---

### Execution flow:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.dialog` | 弹窗容器（影响执行流，需手动触发） | 见Layouts and containers中`st.dialog` |
| `st.form` | 表单容器（批量提交，阻断即时执行） | 见Layouts and containers中`st.form` |
| `st.fragment` | 生成片段（独立刷新，不触发全页重跑） | `@st.fragment` <br> `def refresh_part():` <br> `    st.write(f"当前时间：{datetime.now()}")` <br> `refresh_part()` <br> `st.button("刷新片段", on_click=refresh_part)` |
| `st.rerun` | 强制重新运行应用（刷新页面） | `if st.button("刷新页面"):` <br> `    st.rerun()` |
| `st.stop` | 终止应用执行（后续代码不运行） | `if not st.checkbox("同意条款"):` <br> `    st.stop()` <br> `st.write("继续执行...")` |

---

### Caching and state:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.cache_data` | 缓存数据计算结果（如API请求、数据加载） | `@st.cache_data` <br> `def load_data():` <br> `    return pd.read_csv("large_data.csv")` <br> `df = load_data()` |
| `st.cache_resource` | 缓存资源对象（如数据库连接、模型实例） | `@st.cache_resource` <br> `def init_db():` <br> `    return sqlite3.connect("data.db")` <br> `conn = init_db()` |
| `st.session_state` | 会话状态（跨重跑保存变量） | `if "count" not in st.session_state:` <br> `    st.session_state.count = 0` <br> `if st.button("+1"):` <br> `    st.session_state.count += 1` <br> `st.write("计数：", st.session_state.count)` |
| `st.context` | 获取当前执行上下文（高级用法） | `ctx = st.context` <br> `st.write(f"当前页面：{ctx.page_name}")` |
| `st.query_params` | 读写URL查询参数（跨页面传参） | `# 设置参数` <br> `st.query_params["id"] = 123` <br> `# 获取参数` <br> `id = st.query_params.get("id", "默认值")` |

---

### Connections and secrets:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `st.secrets` | 读取secrets.toml中的敏感信息（如密钥、密码） | `# secrets.toml中：db_password = "123456"` <br> `password = st.secrets["db_password"]` |
| `secrets.toml` | 存储敏感信息的配置文件（非API，配套文件） | 路径：`.streamlit/secrets.toml` <br> 格式：`db_user = "admin"` <br> `db_pass = "123456"` |
| `st.connection` | 简化数据库/服务连接（统一连接管理） | `conn = st.connection("snowflake")` <br> `df = conn.query("SELECT * FROM table LIMIT 10")` |
| `SnowflakeConnection` | Snowflake数据库连接类（st.connection的实现） | `conn = st.connection("snowflake", type=SnowflakeConnection)` |
| `SQLConnection` | SQL数据库通用连接类（支持PostgreSQL/MySQL等） | `conn = st.connection("mysql", type=SQLConnection)` |
| `BaseConnection` | 自定义连接的基类（扩展st.connection） | `class MyConnection(BaseConnection):` <br> `    def _connect(self, **kwargs):` <br> `        return my_client.connect(** kwargs)` <br> `conn = st.connection("my_service", type=MyConnection)` |
| `SnowparkConnection` | Snowpark连接类（Snowflake的Python API） | `conn = st.connection("snowpark", type=SnowparkConnection)` |

---

### Custom components:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `component` | 自定义组件基类（旧版） | （高级用法，需前端开发） <br> `import streamlit.components.v1 as components` <br> `my_component = components.declare_component("my_comp", path="frontend/build")` |
| `ComponentRenderer` | 组件渲染器（旧版） | 配合自定义组件使用，处理渲染逻辑 |
| `component-v2-lib` | 新版自定义组件开发库（更简洁的API） | 安装：`pip install streamlit-component-v2-lib` <br> 开发前端+后端交互组件 |
| `FrontendRenderer` | 前端渲染器（新版组件） | 处理自定义组件的前端渲染参数 |
| `FrontendRendererArgs` | 前端渲染器参数类型（类型提示） | `def render(args: FrontendRendererArgs):` <br> `    return args.components.html("Hello")` |
| `FrontendState` | 前端状态管理（新版组件） | 同步前端与后端的组件状态 |
| `CleanupFunction` | 组件清理函数（释放资源） | `def cleanup():` <br> `    pass` <br> `renderer.set_cleanup_function(cleanup)` |
| `declare_component` | 声明自定义组件（旧版核心函数） | 见`component`示例 |
| `html` | 渲染HTML组件（同st.html，组件内使用） | 见Text elements中`st.html` |
| `iframe` | 嵌入iframe组件（同st.iframe，组件内使用） | 见Text elements中`st.iframe` |

---

### Configuration:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `config.toml` | 应用配置文件（全局设置） | 路径：`.streamlit/config.toml` <br> 示例：`[server]` <br> `port = 8501` <br> `[theme]` <br> `primaryColor = "#ff4b4b"` |
| `st.get_option` | 获取配置项的值 | `port = st.get_option("server.port")` <br> `st.write("当前端口：", port)` |
| `st.set_option` | 动态设置配置项（运行时） | `st.set_option("client.showSidebarNavigation", False)` |
| `st.set_page_config` | 设置页面基础配置（标题、图标、布局） | `st.set_page_config(` <br> `    page_title="我的应用",` <br> `    page_icon="📊",` <br> `    layout="wide"` <br> `)` |

---

### App testing:
| API | 功能 | 基础用法 |
|-----|------|----------|
| `AppTest` | 应用测试类（模拟用户交互） | `from streamlit.testing.v1 import AppTest` <br> `at = AppTest.from_file("app.py").run()` <br> `assert not at.exception` <br> `at.button("点击我").click().run()` <br> `assert at.text[0].value == "按钮被点击了！"` |
| `element_tree` | 元素树（遍历/查询应用UI元素） | `at = AppTest.from_file("app.py").run()` <br> `# 查询所有按钮` <br> `buttons = at.get("button")` <br> `assert len(buttons) == 1` |

---

### Command line:
| 命令 | 功能 | 基础用法 |
|-----|------|----------|
| `streamlit cache` | 缓存管理（清理/查看缓存） | `streamlit cache clear  # 清理所有缓存` <br> `streamlit cache show  # 查看缓存信息` |
| `streamlit config` | 配置管理（查看/编辑配置） | `streamlit config show  # 查看所有配置项` <br> `streamlit config set server.port=8502  # 设置配置` |
| `streamlit docs` | 打开Streamlit官方文档 | `streamlit docs` |
| `streamlit hello` | 运行官方示例应用 | `streamlit hello` |
| `streamlit help` | 查看命令帮助 | `streamlit help run  # 查看run命令帮助` |
| `streamlit init` | 初始化Streamlit项目（生成配置/目录） | `streamlit init my_app  # 创建my_app项目` |
| `streamlit version` | 查看Streamlit版本 | `streamlit version` |

---

### 一点点补充
1. 所有示例均基于Streamlit 1.32.0版本，部分API（如`st.logo`、`st.fragment`）为新版特性，低版本可能不支持；
2. 基础用法仅展示核心场景，更多参数（如样式、回调函数、自定义配置）可参考[Streamlit官方文档](https://docs.streamlit.io/)；
3. 自定义组件、数据库连接等高级API需结合具体场景扩展，建议配合官方示例项目学习。