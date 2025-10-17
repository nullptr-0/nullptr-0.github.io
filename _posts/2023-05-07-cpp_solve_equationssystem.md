---
layout: post
title: "多元一次非齐次方程组求解（C++描述）"
date:   2023-05-07
tags: [Linear Algebra,C++,Math]
comments: true
author: nullptr-0
---

在网上翻了很久都没有找到可以处理包含自由变量的实现，索性就自己手搓了一个。<br/><br/>记录一下：

```C++
#include <iostream>
#include <vector>

using namespace std;

// 高斯消元法求解方程组
vector<vector<double>> solveEquations(vector<vector<double>>& equations) {
    int n = equations.size(); // 方程组的行数
    int m = equations[0].size() - 1; // 方程组的列数（未知数的个数）
    if (m != n) {
        return vector<vector<double>>();
    }

    // 将方程组转化为上三角矩阵
    for (int i = 0; i < n - 1; i++) {
        // 寻找主元（绝对值最大的元素）
        int maxRow = i;
        for (int j = i + 1; j < n; j++) {
            if (abs(equations[j][i]) > abs(equations[maxRow][i])) {
                maxRow = j;
            }
        }

        // 交换当前行和主元所在行
        if (maxRow != i) {
            swap(equations[i], equations[maxRow]);
        }

        // 确定主元
        double pivot = equations[i][i];
        if (abs(pivot) < 1e-10) {
            // 主元为0，跳出
            break;
        }

        // 消元
        for (int j = i + 1; j < n; j++) {
            double ratio = equations[j][i] / pivot;
            for (int k = i; k <= m; k++) {
                equations[j][k] -= ratio * equations[i][k];
            }
        }
    }

    vector<bool> freeVariables(m, true);
    // 回代求解
    vector<vector<double>> solution(m, vector<double>(m + 1, 0.0));
    for (int i = m - 1; i >= 0; i--) {
        if (abs(equations[i][i]) < 1e-10) {
            // 对角元素为0，方程组无解或有无穷解
            bool hasSolution = true;
            for (int j = 0; j < i; j++) {
                if (abs(equations[i][j]) > 1e-10) {
                    hasSolution = false;
                    break;
                }
            }
            if (hasSolution) {
                solution[i][i + 1] = 1;
            }
            else {
                return vector<vector<double>>();
            }
            hasSolution = false;
            int i2 = i + 1;
            for (; i2 < m; ++i2) {
                if (abs(equations[i][i2]) > 1e-10) {
                    hasSolution = true;
                    break;
                }
            }
            if (hasSolution) {
                freeVariables[i2] = false;
                solution[i2][i2 + 1] = 0;
                solution[i2][0] = equations[i][m] / equations[i][i2];
                for (int j = i2 + 1; j < m; ++j) {
                    for (int k = 0; k < m + 1; ++k)
                    {
                        solution[i2][k] -= equations[i][j] * solution[j][k] / equations[i][i2];
                    }
                }
                for (int j = i + 1; j < m; ++j)
                {
                    if (solution[j][i2 + 1] != 0)
                    {
                        for (int k = 0; k < m + 1; ++k)
                        {
                            solution[j][k] += solution[j][i2 + 1] * solution[i][k];
                        }
                        solution[j][i2 + 1] = 0;
                    }
                }
            }
            continue;
        }
        freeVariables[i] = false;
        solution[i][i + 1] = 0;
        solution[i][0] = equations[i][m] / equations[i][i];
        for (int j = i + 1; j < m; ++j) {
            for (int k = 0; k < m + 1; ++k)
            {
                solution[i][k] -= equations[i][j] * solution[j][k] / equations[i][i];
            }
            if (solution[j][i + 1] != 0)
            {
                for (int k = 0; k < m + 1; ++k)
                {
                    solution[j][k] += solution[j][i + 1] * solution[i][k];
                }
                solution[j][i + 1] = 0;
            }
        }
    }

    return solution;
}

int main() {
    int n; // 方程组的行数
    int m; // 方程组的列数（未知数的个数）
    cout << "请输入方程组的行数和列数（以空格分隔）：";
    cin >> n >> m;
    int order = max(m, n);

    vector<vector<double>> equations(order, vector<double>(order + 1, 0.0));
    cout << "请输入方程组的增广矩阵（元素以空格分隔）：" << endl;
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < m + 1; j++) {
            cin >> equations[i][j];
        }
    }

    auto solution = solveEquations(equations);

    if (!solution.size())
    {
        cout << "方程组无解" << endl;
    }

    cout << "方程组的通解为：" << endl;
    for (const auto& row : solution)
    {
        for (const auto& el : row)
        {
            cout << el << ",\t";
        }
        cout << endl;
    }

    return 0;
}
```
