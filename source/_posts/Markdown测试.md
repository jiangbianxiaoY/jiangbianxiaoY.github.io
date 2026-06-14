---
title: Markdown 渲染功能测试
date: 2026-06-14 18:00:00
updated: 2026-06-14 18:00:00
categories:
  - 测试
tags:
  - Markdown
  - 测试
  - 渲染
---

## 一、标题

# H1
## H2
### H3
#### H4
##### H5
###### H6

## 二、文本样式

普通文本 **加粗** *斜体* ~~删除线~~ `行内代码` :smile: :rocket: :+1:

下标: H~2~O   上标: E=mc^2^ (markdown-it-sub / sup)

## 三、代码块

JavaScript:
```javascript
function greeting(name) {
  const msg = `Hello, ${name}!`;
  console.log(msg);
  return msg;
}

// 箭头函数
const add = (a, b) => a + b;
```

Python:
```python
import numpy as np

class NeuralNetwork:
    def __init__(self, layers):
        self.layers = layers
        self.params = {}

    def forward(self, x):
        for layer in self.layers:
            x = layer(x)
        return x

model = NeuralNetwork([64, 128, 10])
```

Bash:
```bash
#!/bin/bash
for i in {1..10}; do
  echo "Count: $i"
done
```

没有语言标识的代码块:
```
纯文本代码块
第二行
```

## 四、表格

### 4.1 基本表格

| 名称 | 版本 | 说明 |
|------|------|------|
| Hexo | 8.0.0 | 博客框架 |
| Redefine | 2.8.5 | 主题 |
| Node.js | 22.x | 运行时 |

### 4.2 对齐表格

| 左对齐 | 居中对齐 | 右对齐 |
|:-------|:--------:|-------:|
| 左 | 中 | 右 |
| 苹果 | 香蕉 | 樱桃 |
| 100 | 200 | 300 |

### 4.3 三线表测试

三线表样式：顶部 2px 线、表头下方 2px 线、底部 2px 线，无竖线。

| 学号 | 姓名 | 数学 | 物理 | 英语 |
|:----|:----|:----|:----|:----|
| 001 | 张三 | 95 | 88 | 92 |
| 002 | 李四 | 87 | 91 | 85 |
| 003 | 王五 | 78 | 82 | 90 |

长表格测试：

| 项目 | 描述 | 优先级 | 状态 | 负责人 | 截止日期 |
|:----|:----|:----:|:----|:----|:----|
| 博客搭建 | 恢复 Hexo 博客系统 | P0 | ✅ 已完成 | 江边小余 | 2026-06-14 |
| 主题配置 | 安装 Redefine 主题 | P0 | ✅ 已完成 | 江边小余 | 2026-06-14 |
| 文章恢复 | 恢复 ResNet 文章 | P1 | ✅ 已完成 | 江边小余 | 2026-06-15 |
| 渲染修复 | 修复 MathJax 公式渲染 | P1 | ✅ 已完成 | 江边小余 | 2026-06-14 |
| 表格样式 | 添加三线表 CSS | P2 | ✅ 已完成 | 江边小余 | 2026-06-14 |
| 评论系统 | 集成评论功能 | P2 | ❌ 待完成 | - | - |
| SEO优化 | 搜索引擎优化 | P3 | ❌ 待完成 | - | - |

### 4.4 含代码和格式的表格

| 语法 | 示例 | 说明 |
|:-----|:----|:-----|
| `**bold**` | **加粗** | 粗体字 |
| `*italic*` | *斜体* | 斜体字 |
| `` `code` `` | `code` | 行内代码 |
| `~~text~~` | ~~删除~~ | 删除线 |

## 五、列表

### 5.1 无序列表

- 苹果
- 香蕉
- 樱桃
  - 红樱桃
  - 黑樱桃
- 榴莲

### 5.2 有序列表

1. 第一步：安装 Hexo
2. 第二步：配置主题
3. 第三步：编写文章
4. 第四步：部署上线

### 5.3 任务列表

- [x] 完成博客搭建
- [x] 恢复 ResNet 文章
- [ ] 配置评论系统
- [ ] 添加访问统计
- [ ] 优化 SEO

### 5.4 混合列表

1. 准备工作
   - 安装 Node.js
   - 安装 Git
2. 项目初始化
   - `hexo init blog`
   - `cd blog`
3. 配置主题
   - [x] 下载 Redefine
   - [ ] 自定义样式

## 六、引用

### 6.1 单层引用

> 学而不思则罔，思而不学则殆。

### 6.2 多层引用

> 第一层引用
>
> > 第二层引用
> >
> > > 第三层引用

### 6.3 引用内嵌其他元素

> ## 引用内的标题
>
> - 列表项一
> - 列表项二
>
> `code` **加粗** *斜体*

## 七、链接

### 7.1 普通链接

[Hexo 官网](https://hexo.io)

[Redefine 主题](https://github.com/EvanNotFound/hexo-theme-redefine)

### 7.2 带标题的链接

[GitHub](https://github.com "全球最大代码托管平台")

### 7.3 自动链接

<https://hexo.io>

<admin@example.com>

## 八、脚注

这是一段需要注释的文字[^1]，这里还有一段[^2]。

[^1]: 这是第一条脚注的内容。
[^2]: 这是第二条脚注，可以包含多行内容。
    第二行内容也在脚注中。

## 九、定义列表（Description List）

Hexo
: 一个快速、简洁且高效的博客框架，基于 Node.js。

Redefine
: Hexo 的一款现代化主题，支持丰富的自定义配置。

Markdown
: 一种轻量级标记语言，使用纯文本格式编写文档。

## 十、缩写（Abbreviation）

HTML 是超文本标记语言。

CSS 用于样式设计。

*[HTML]: HyperText Markup Language
*[CSS]: Cascading Style Sheets

## 十一、LaTeX 数学公式

### 11.1 行内公式

欧拉公式: $e^{i\pi} + 1 = 0$

勾股定理: $a^2 + b^2 = c^2$

### 11.2 块级公式

$$
\dfrac{\partial u}{\partial t} = \alpha \left( \dfrac{\partial^2 u}{\partial x^2} + \dfrac{\partial^2 u}{\partial y^2} + \dfrac{\partial^2 u}{\partial z^2} \right)
$$

Softmax 函数:

$$
\sigma(\mathbf{z})_i = \frac{e^{z_i}}{\sum_{j=1}^{K} e^{z_j}}
$$

### 11.3 表格内数学公式

| 公式 | 名称 | 用途 |
|:----|:----|:----|
| $\mu = \frac{1}{N}\sum_{i=1}^{N} x_i$ | 均值 | 中心趋势 |
| $\sigma^2 = \frac{1}{N}\sum_{i=1}^{N}(x_i - \mu)^2$ | 方差 | 离散程度 |
| $\text{Cov}(X,Y) = \mathbb{E}[(X-\mu_X)(Y-\mu_Y)]$ | 协方差 | 相关性 |

## 十二、Emoji

:heart: :fire: :star: :zap: :sunny: :moon: :cloud: :rainbow:

:dog: :cat: :panda_face: :frog: :butterfly:

:+1: :-1: :clap: :raised_hands: :pray:

## 十三、分割线

---

***

___

## 十四、图片

![图片测试](images/head.jpg "博客头像")

## 十五、HTML 内联元素

<kbd>Ctrl</kbd> + <kbd>C</kbd> 复制

<kbd>Ctrl</kbd> + <kbd>V</kbd> 粘贴

<mark>高亮文本</mark> 和 <del>删除文本</del>

## 十六、复杂混合测试

### 16.1 代码 + 表格 + 引用

> 查看以下代码和表格:
>
> ```python
> def fibonacci(n):
>     a, b = 0, 1
>     for _ in range(n):
>         yield a
>         a, b = b, a + b
> ```
>
> | n | 结果 |
> |:-:|:----|
> | 5 | 0,1,1,2,3 |
> | 8 | 0,1,1,2,3,5,8,13 |

### 16.2 列表 + 代码 + 公式

1. **线性回归**
   - 公式: $y = wx + b$
   - 代码:
     ```python
     y = w * x + b
     loss = (y - y_true) ** 2
     ```
   - 损失函数: $\mathcal{L} = \frac{1}{N}\sum_{i=1}^{N}(y_i - \hat{y}_i)^2$

2. **逻辑回归**
   - 公式: $\sigma(z) = \frac{1}{1 + e^{-z}}$
   - 决策边界: $P(y=1|x) > 0.5$

### 16.3 任务列表 + 表格

| 优先级 | 任务 | 状态 |
|:------|:----|:----|
| 高 | 配置 CI/CD | - [x] 已完成 |
| 中 | 添加评论 | - [ ] 待完成 |
| 低 | 主题升级 | - [ ] 待完成 |

## 十七、结论

以上涵盖了 Markdown 的主流渲染语法:
- 基础样式 ✓
- 代码块 ✓
- 表格（含三线表） ✓
- 列表 ✓
- 引用 ✓
- 链接 ✓
- 数学公式 ✓
- Emoji ✓
- 脚注 ✓
- 定义列表 ✓
- 上标/下标 ✓
