# bit树
## LSB
### 什么是LSB
LSB全称"Least Significant Bit"，即“最低有效位”，也就是数字二进制中从低位开始的第一个1在哪一位
### 代码
```c++
int LSB(int a){
	return a & (-a);
} 
/*代码返回的是最低有效位连带后面的0组成的数
如8的二进制为1000，那么函数返回的就是8 ，即二进制的1000
*/
```
## bit树
### 什么是bit树
bit树状数组可以解决连续和、前缀和的动态查询.
bit树需要一个数组，这里叫bit
那么bit[i]也就是以i结尾，长度为LSB（i） 的区间和。
（为什么是结尾呢，因为你的区间长度就是结尾编号的LSB，就像前缀和一样）
画出来可以发现：
- 你可以用这些区间快速拼凑出所有从区间开始处开始的区间（前缀和）

- 每个区间的右侧区间都没有，因为有左区间就够了。

---
### 查询
他的查询思路就是用多个区间和相加的方式拼凑出前缀和，再通过前缀和从而求出区间 
我们现在定义一个函数，叫`psq(i)`，可以求出区间[1,i]的区间和(其实就是前缀和),
那么我们要求[j,k] 就是 `psq(k) - psq(j-i) ` (区间和的思路)
那`psq(i)`如何实现呢。例如要查区间[1,i]
就要让i不断的减`LSB(i)` , i 在变小，LSB参数中的i也在变小 ，然后减到0
此时过程中i会出现许多数，只要每次减完LSB后把bit[i]的值累加起来就可以了
因为bit[i]代表的区间的长度就是`LSB(i)`,所以减完LSB后的到的区间就是以原区间的左端点编号-1得到的

---
### 更新

他比前缀和好的一点是每次更新不需要更新这个位置和后面的所有前缀和
例如我要更新点i , 值为z，那么我只需不断的像查寻那样，不过这里是加`LSb[i]`,接着把每个bit[i]都更新一下，直到i超过数组的长度（题目里给的数组的长度，就是数组哪些位置是有效的，有有意义的数字的）
### 代码
```c++
#include<bits/stdc++.h>
using namespace std;
const int N = 1e5;
int bit[N], n;
int LSB(int a){
	return a & (-a);
} 
//查询 
long long psq(long long i){
	long long sum = 0;
	while(i){//i!=0
		sum += bit[i];
		i -= LSB(i);
	}	
	return sum;
}
//添加 
void add(long long i, long long z){
	// i:更新位置，z:增加的值 
	while(i <= n){
		bit[i] += z;
		i += LSB(i);
	} 
}
int main() {
	int m;
	cin >> n >> m;
	for (int i = 1;i <= n ; i++){
		int t;
		cin >> t;
		add(i,t);
	} 
	while(m--){
		int l, r;
		cin >> l >> r;
		cout << psq(r) - psq(l-1) << endl;
	}
	return 0;
}


```
参考太戈编程课件，侵删。
有错请指出。
