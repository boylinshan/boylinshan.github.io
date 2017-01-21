---
layout: post
title: "Searching a triangular array for a sub-triangle having minimum-sum"
category: C++
---

# Searching a triangular array for a sub-triangle having minimum-sum

In a triangular array of positive and negative integers, we wish to find a subtriangle such that the sum of the numbers it contains is the smallest possible.

In the example below, it can be easily verified that the marked triangle satisfies this condition having a sum of **-42**.

Now, let's extend the problem. Suppose we have such a triangular array with **N** rows, thus there are  $\frac{N(N+1)}{2}$ entries all in all.

Subtriangles can start at any element of the array and extend down as far as we like (taking-in the two elements directly below it from the next row, the three elements directly below from the row after that, and so on).

The "sum of a subtriangle" is defined as the sum of all the elements it contains.

Given **K**, find the **K** smallest possible subtriangle sums. Consider all subtriangles distinct, even though some of them may have the same sums.

#### Input Format

The first line of input contains two integers **N** and  **K** separated by a space.

The next  **N** lines contain the entries of the triangle. Specifically, the $$\imath$$ th following line contains  $\imath$ integers, denoting the entries in the $\imath$ th row of the triangle.

#### Constraints
- $$1 \leq N \leq 350$$
- $-10^5 \leq triangle \; entries \leq 10^5$
- $K \geq 1$

#### Output Format
Output **K** lines. The $\imath$th line contains the $\imath$th smallest subtriangle sum.

#### Sample Input
```
6 5
15
-14 -7
20 -13 -5
-3 8 23 -26
1 -4 -5 -18 5
-16 31 2 9 28 3
```

#### Sample Output

```
-42
-39
-26
-26
-25
```

#### Solution
```cpp
void heapify(vector<int> &vec, int pos, int n) {
	int largest = pos;
	int right = 2 * largest + 1;
	int left = 2 * largest+ 2;

	if (right<n && vec[right] > vec[largest])
		largest = right;
	if (left<n && vec[left] > vec[largest])
		largest = left;
	if (largest != pos) {
		swap(vec[largest], vec[pos]);
		heapify(vec, largest, n);
	}
}

vector<int> Solution(vector<vector<int>> &matrix, int K) {
	vector<vector<vector<int>>> sum;
	sum.resize(matrix.size());
	for (int i = 0; i < matrix.size(); ++i) {
		sum[i].resize(matrix[i].size());
		for (int j = 0; j <= i; ++j) {
			sum[i][j].resize(matrix.size() - i);
			sum[i][j][0] = matrix[i][j];
		}
	}
	
	#求每一个三角行的和
	for (int i = sum.size() - 2; i >= 0; --i) {
		for (int j = 0; j < sum[i].size(); ++j) {
			for (int k = 1; k < sum[i][j].size(); ++k) {
				sum[i][j][k] = sum[i][j][0] + sum[i + 1][j][k - 1] + sum[i + 1][j + 1][k - 1];
				if (i+2<sum.size() && k-2>=0) {
					sum[i][j][k] -= sum[i + 2][j + 1][k-2];
				}	
				
			}
		}
	}
	
	#堆排序，仅保留最小的K个
	vector<int> result(K, INT_MAX);
	for (int i = 0; i < sum.size(); ++i) {
		for (int j = 0; j < sum[i].size(); ++j) {
			for (int k = 0; k < sum[i][j].size(); ++k) {
				int elem = sum[i][j][k];
				result[0] = elem < result[0] ? elem : result[0];
				heapify(result,0, result.size());
			}
		}
	}

	sort(result.begin(), result.end());

	return result;
}
```

最直接的解法，求出所有三角行的合，再选出最小的**K**个。
求合时，考虑到对于以 $a_{i,j}$为顶点，高度为**n**的三角形而言，可由分别以 $a_{i+1,} \ a_{i+1,j+i} \ a_{i+2,j+2}$为顶点，高度为**n-1, n-1, n-2**的三角形求出，即以 $a_{i,j}$为顶点的棱形的4个顶点。所以可以采用动态规划的方法，从最低端开始，减少重复的计算。

此外在进行筛选合时，采用堆排序的方式，减少比较的次数。

				 
