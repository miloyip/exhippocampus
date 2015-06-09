---
layout: page
title: 如何用 C 语言画一个心形
categories:
    - graphics
---

## 基础版

![draw_heart01](/images/draw_heart01.jpg)

这版本使用了一个函数 [heart curve](http://mathworld.wolfram.com/HeartCurve.html)：

$$
f(x, y) = (x^2 + y^2 - 1)^3 - x^2y^3
$$

~~~C
#include <stdio.h>

int main() {
    for (float y = 1.5f; y > -1.5f; y -= 0.1f) {
        for (float x = -1.5f; x < 1.5f; x += 0.05f) {
            float a = x * x + y * y - 1;
            putchar(a * a * a - x * x * y * y * y <= 0.0f ? '*' : ' ');
        }
        putchar('\n');
    }
}
~~~

## 花纹版

![draw_heart02](/images/draw_heart02.jpg)

这其实是该函数的 [level set](http://en.wikipedia.org/wiki/Level_set)。

~~~C
#include <stdio.h>

int main() {
    for (float y = 1.5f; y > -1.5f; y -= 0.1f) {
        for (float x = -1.5f; x < 1.5f; x += 0.05f) {
            float z = x * x + y * y - 1;
            float f = z * z * z - x * x * y * y * y;
            putchar(f <= 0.0f ? ".:-=+*#%@"[(int)(f * -8.0f)] : ' ');
        }
        putchar('\n');
    }
}
~~~

## 3D版

![draw_heart03](/images/draw_heart03.jpg)

[Heart surface](http://mathworld.wolfram.com/HeartSurface.html) 是一个隐含函数的等值面：

$$
\left (x^2 + \frac{9}{4} y^2 z^2 - 1 \right)^3 - x^2 z^3 - \frac{9}{80} y^2 z^3 = 0
$$

这个实现简单使用迭代法求出每个$(x, y)$的$z$，然后用[finite difference](http://en.wikipedia.org/wiki/Finite_difference)求法矢量，用wrapped diffuse着色。

~~~C
#include <stdio.h>
#include <math.h>

float f(float x, float y, float z) {
    float a = x * x + 9.0f / 4.0f * y * y + z * z - 1;
    return a * a * a - x * x * z * z * z - 9.0f / 80.0f * y * y * z * z * z;
}

float h(float x, float z) {
    for (float y = 1.0f; y >= 0.0f; y -= 0.001f)
        if (f(x, y, z) <= 0.0f)
            return y;
    return 0.0f;
}

int main() {
    for (float z = 1.5f; z > -1.5f; z -= 0.05f) {
        for (float x = -1.5f; x < 1.5f; x += 0.025f) {
            float v = f(x, 0.0f, z);
            if (v <= 0.0f) {
                float y0 = h(x, z);
                float ny = 0.01f;
                float nx = h(x + ny, z) - y0;
                float nz = h(x, z + ny) - y0;
                float nd = 1.0f / sqrtf(nx * nx + ny * ny + nz * nz);
                float d = (nx + ny - nz) * nd * 0.5f + 0.5f;
                putchar(".:-=+*#%@"[(int)(d * 5.0f)]);
            }
            else
                putchar(' ');
        }
        putchar('\n');
    }
}
~~~

## 彩色版

![draw_heart04](/images/draw_heart04.jpg)

把「3D版」输出至 PPM 文件，可以用 Photoshop 打开。另外降低了`ny`的值导致有超有趣的 pattern，就保留下来吧。

~~~C
#ifdef _MSC_VER
#define _CRT_SECURE_NO_WARNINGS
#endif
#include <stdio.h>
#include <math.h>

float f(float x, float y, float z) {
    float a = x * x + 9.0f / 4.0f * y * y + z * z - 1;
    return a * a * a - x * x * z * z * z - 9.0f / 80.0f * y * y * z * z * z;
}

float h(float x, float z) {
    for (float y = 1.0f; y >= 0.0f; y -= 0.001f)
        if (f(x, y, z) <= 0.0f)
            return y;
    return 0.0f;
}

int main() {
    FILE* fp = fopen("heart.ppm", "w");
    int sw = 512, sh = 512;
    fprintf(fp, "P3\n%d %d\n255\n", sw, sh);
    for (int sy = 0; sy < sh; sy++) {
        float z = 1.5f - sy * 3.0f / sh;
        for (int sx = 0; sx < sw; sx++) {
            float x = sx * 3.0f / sw - 1.5f;
            float v = f(x, 0.0f, z);
            int r = 0;
            if (v <= 0.0f) {
                float y0 = h(x, z);
                float ny = 0.001f;
                float nx = h(x + ny, z) - y0;
                float nz = h(x, z + ny) - y0;
                float nd = 1.0f / sqrtf(nx * nx + ny * ny + nz * nz);
                float d = (nx + ny - nz) / sqrtf(3) * nd * 0.5f + 0.5f;
                r = (int)(d * 255.0f);
            }
            fprintf(fp, "%d 0 0 ", r);
        }
        fputc('\n', fp);
    }
    fclose(fp);
}
~~~

## ASCII动画

通过空间的缩放变换实现 ASCII 心跳动画，需要改变光标位置，此版本仅支持 Windows。录制视频不太顺畅，建议在本地测试。

~~~C
#include <stdio.h>
#include <math.h>
#include <windows.h>
#include <tchar.h>

float f(float x, float y, float z) {
    float a = x * x + 9.0f / 4.0f * y * y + z * z - 1;
    return a * a * a - x * x * z * z * z - 9.0f / 80.0f * y * y * z * z * z;
}

float h(float x, float z) {
    for (float y = 1.0f; y >= 0.0f; y -= 0.001f)
        if (f(x, y, z) <= 0.0f)
            return y;
    return 0.0f;
}

int main() {
    HANDLE o = GetStdHandle(STD_OUTPUT_HANDLE);
    _TCHAR buffer[25][80] = { _T(' ') };
    _TCHAR ramp[] = _T(".:-=+*#%@");

    for (float t = 0.0f;; t += 0.1f) {
        int sy = 0;
        float s = sinf(t);
        float a = s * s * s * s * 0.2f;
        for (float z = 1.3f; z > -1.2f; z -= 0.1f) {
            _TCHAR* p = &buffer[sy++][0];
            float tz = z * (1.2f - a);
            for (float x = -1.5f; x < 1.5f; x += 0.05f) {
                float tx = x * (1.2f + a);
                float v = f(tx, 0.0f, tz);
                if (v <= 0.0f) {
                    float y0 = h(tx, tz);
                    float ny = 0.01f;
                    float nx = h(tx + ny, tz) - y0;
                    float nz = h(tx, tz + ny) - y0;
                    float nd = 1.0f / sqrtf(nx * nx + ny * ny + nz * nz);
                    float d = (nx + ny - nz) * nd * 0.5f + 0.5f;
                    *p++ = ramp[(int)(d * 5.0f)];
                }
                else
                    *p++ = ' ';
            }
        }

        for (sy = 0; sy < 25; sy++) {
            COORD coord = { 0, sy };
            SetConsoleCursorPosition(o, coord);
            WriteConsole(o, buffer[sy], 79, NULL, 0);
        }
        Sleep(33);
    }
}
~~~

## Shadertoy

![draw_heart06](/images/draw_heart06.jpg)

移植至 [Shadertoy](https://www.shadertoy.com/view/XtXGR8)，加入高光和背景。

本文原发表于[知乎](http://www.zhihu.com/question/20187195/answer/34873279)，经过修改。