---
layout: post
read_time: true
show_date: true
title:  "Data structure"
date:   2021-03-12 13:32:20 -0600
description: Learning data struct makes you stronger.
img: posts/20210125/Perceptron.jpg 
tags: [coding, java, c++, go, data struct]
author: TM
github:  https://github.com/Tomxiaobai
mathjax: yes
---
## 数据结构
## 前缀和

- 二维前缀和 [LeetCode-304](https://leetcode.cn/problems/range-sum-query-2d-immutable/)
- 题目描述：给定一个二维矩阵 matrix，以下类型的多个请求：计算其子矩形范围内元素的总和，该子矩阵的 左上角 为 (row1, col1) ，右下角为 (row2, col2)。
    实现 NumMatrix 类： NumMatrix(int[][] matrix) 给定整数矩阵 matrix 进行初始化 int sumRegion(int row1, int col1, int row2, int col2) 返回 左上角 (row1, col1) 、右下角 (row2,col2) 所描述的子矩阵的元素总和。
- 和一维前缀和相同，首先定义一个求和数组，其每个元素代表从（0， 0）到 （x, y）的元素总和。可以按照官方题解给出的图进行理解。
<center><img src='./assets/img/posts/20220414/leetcode_304.png'></center>
* 因此对应的求(row1, col1, row2, col2)的元素和可以通过下图进行理解。
<center><img src='.assets/img/posts/20220414/two_matrix.png'></center>

示例Demo:
```java
public class NumMatrix {
    // 二维前缀和模块例题
    private int[][] sum;
    public NumMatrix(int[][] nums) {
        int n = nums.length;
        int m = n == 0 ? 0 : nums[0].length;
        sum = new int[n+1][m+1];
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < m; j++) {
                sum[i+1][j+1] = sum[i][j+1] + sum[i+1][j] - sum[i][j] + nums[i][j];
            }
        }
    }
    public int sumRegion(int row1, int col1, int row2, int col2) {
        return sum[row2+1][col2+1] - sum[row1][col2+1] - sum[row2+1][col1] + sum[row1][col1];
    }
}
```





    