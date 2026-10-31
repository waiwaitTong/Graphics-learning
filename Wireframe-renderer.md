# TinyRenderer 学习记录：从 OBJ 到线框图

TinyRenderer 学习画线，用 C++ 读取 OBJ 模型，画出了第一张线框图。代码在 GPT 的帮助下逐步实现、排错。

## 1. 总体思路

读取 OBJ → 保存顶点和面 → 三维坐标投影到二维 → 转换成像素坐标 → 画每个三角形的三条边 → 保存图片。

本次先沿 z 轴观察，舍去 z、保留 x 和 y，相当于最简单的正交投影。暂时把所有边都画出来，不处理遮挡。

## 2. OBJ 怎样读进程序

OBJ 是文本文件，当前只处理两种行：

```text
v 0.11526 0.700717 0.0677257
f 1193/1240/1193 1180/1227/1180 1179/1226/1179
```

`v` 后面是顶点的三维坐标；`f` 后面每组数据依次是“顶点 / 纹理坐标 / 法线”的索引。本次只取每组的第一个数。OBJ 的正索引从 1 开始，存进 C++ 下标时减一。

用两张表分别保存坐标与连接关系：

```cpp
struct Vertex { float x, y, z; };
struct Face { int a, b, c; };

std::vector<Vertex> vertices;
std::vector<Face> faces;
```

下面是读取的核心片段，放在 `load_obj()` 中；需要 `<fstream>`、`<sstream>`、`<string>`、`<vector>` 和 `<iostream>`。

```cpp
std::ifstream file("./obj/diablo3_pose/diablo3_pose.obj");
if (!file) {
    std::cerr << "Cannot open model file\n";
    return;
}

std::string text;
while (std::getline(file, text)) {
    std::istringstream stream(text);
    std::string type;
    stream >> type;

    if (type == "v") {
        Vertex v{};
        if (stream >> v.x >> v.y >> v.z)
            vertices.push_back(v);
    } else if (type == "f") {
        std::string a, b, c;
        if (stream >> a >> b >> c) {
            faces.push_back({std::stoi(a) - 1,
                             std::stoi(b) - 1,
                             std::stoi(c) - 1});
        }
    }
}
```

`getline` 读取整行文字，`istringstream` 用来逐项解析这一行；`>>` 会记住读取位置，并按接收变量的类型解析数据。`stoi("1193/1240/1193")` 在斜杠处停止，得到整数 1193。



## 3. 坐标怎样变成像素

这里假设模型的 x、y 坐标在 `[-1,1]` 内。以横坐标为例：

```cpp
int px = std::round((x + 1.0f) / 2.0f * (width - 1));
```

先把 `[-1,1]` 平移到 `[0,2]`，再缩放到 `[0,1]`，最后映射到 `[0,width-1]`。乘 `width-1`，是因为宽为 `width` 的图片，最后一个像素编号是 `width-1`。纵坐标同理，`std::round` 需要 `<cmath>`。

读取完成后，遍历每个面，从顶点表取出坐标，画出三条边。下面沿用已有的 `line()` 函数、图片对象 `image` 和颜色 `blue`：

```cpp
for (int i = 0; i < faces.size(); ++i) {
    Vertex v0 = vertices[faces[i].a];
    Vertex v1 = vertices[faces[i].b];
    Vertex v2 = vertices[faces[i].c];

    int x0 = std::round((v0.x + 1.0f) * (width - 1) / 2.0f);
    int y0 = std::round((v0.y + 1.0f) * (height - 1) / 2.0f);
    int x1 = std::round((v1.x + 1.0f) * (width - 1) / 2.0f);
    int y1 = std::round((v1.y + 1.0f) * (height - 1) / 2.0f);
    int x2 = std::round((v2.x + 1.0f) * (width - 1) / 2.0f);
    int y2 = std::round((v2.y + 1.0f) * (height - 1) / 2.0f);

    line(x0, y0, x1, y1, image, blue);
    line(x1, y1, x2, y2, image, blue);
    line(x2, y2, x0, y0, image, blue);
}
image.write_tga_file("wireframe.tga");
```

相邻三角形的公共边可能重复绘制，当前先保留这个简单实现。

## 4. 坑：小数增量被丢掉

第一次生成的图片里，大量线条变成了水平、竖直或约 45° 的斜线。问题出在画线函数中：

```cpp
int y = y0;
y += (y1 - y0) / (float)delta_x;
```

右边已经是浮点除法，但每轮写回 `int y` 时，小数仍会被截掉。例如 `20 + 0.3` 存回整数是 20，下一次又从 20 开始；正数坐标下，`20 - 0.3` 则变成 19，可能每次都下降一个像素。

修正方法是让 `y` 保留小数，只在画像素时取整。下面是画线函数完成陡线换轴、端点排序之后的核心循环；零长度线段应提前单独画一个点并返回，避免除以零。

```cpp
float y = y0;
float step = (y1 - y0) / static_cast<float>(x1 - x0);
for (int x = x0; x <= x1; ++x) {
    int py = std::round(y);
    if (steep) image.set(py, x, color);
    else       image.set(x, py, color);
    y += step;
}
```

先画当前点，再增加 `y`，起点才不会提前偏移。这次记住的原则是：**计算位置时保留小数，选取像素时才取整，取整结果不用于下一轮累加。** 当前使用的是浮点增量画线；教程中的误差累积与整数画线还需要继续巩固。

## 5. 其他收获与待补内容

- **编译与运行：**`g++ -o wireframe.exe` 生成程序；运行程序后，`write_tga_file` 才生成图片。把编译产物命名为 `.tga` 不会让它变成图片。
- **文件路径：**`./` 是当前工作目录，`../` 是上一级。相对路径以运行时的工作目录为起点。
- **声明与定义：**若沿用全局顶点表，头文件写 `extern std::vector<Vertex> vertices;`，只在一个 `.cpp` 中定义；使用它的 `.cpp` 也需要包含该头文件。
- **与 AI 协作：**先自己拆问题、写代码，再带着报错和异常图片定位；修复后补上“为什么”，避免只复制修改结果。

目前能生成线框图，接下来要测试不同方向、端点倒置和零长度线段，并复习 C++ 作用域、流与数值转换。工具上还想学习 VS Code 构建任务与 CMake，减少重复输入编译命令。

参考：[TinyRenderer 画线教程](https://haqr.eu/tinyrenderer/bresenham/) · [项目仓库](https://github.com/ssloy/tinyrenderer)。图片读写工具及模型素材沿用原项目。
