# st表
用于解决RMQ(区间最值查询)
## 对数换底公式
当a>0，a≠1，b>0，b≠1且N>0时，
以b为底N的对数 = 以a为底N的对数 / 以a为底b的对数
## 预计算
`st[p][i]`代表从i号开始共 2^p个数的最大值，也就是[i,i+2^p-1]区间的最大值
当p为1时，`st[0][i]` 均为原值
之后的其他行，每个值都由上一行的两覆盖了这的区间的区间拼接而成
## 查询
同样由两个区间拼接而成
先算出区间长度以2为底时的对数（确定在哪一行）
接着由重叠的区间拼接而成的（最大值重叠没有影响）
## 代码
```
#include<bits/stdc++.h>
using namespace std;
const int N = 1e5;
int n , m , l , r;
int x[N] , st[N][20];
void ST(){
	for (int i = 1;i <= n ; i++){
		st[0][i] = x[i];
		//第一行均为原来的值 
	}
	int p = log(n) / log(2);
	//以2为底，n的对数
	for (int k = 1; k <= p ; k++){
		for (int i = 1; i <= n - (1 << k) + 1; i++){
			st[k][i] = max(st[k-1][i],st[k-1][i+(1<<(k-1))]);
			//拼接上一行 
		}
	} 
}
int query(int l , int r){
	int p = log(r-l + 1) / log(2);
	return max(st[p][l],st[p][r-(1<<p)+1]) ;
	// r-(1<<p)+1 由r反推l 
}
int main() {
	cin >> n >> m;
	for (int i = 1; i <= n ; i++){
		cin >> x[i];
	}
	ST();
	for (int i= 1;i <= m ; i++){
		cin >> l >> r;
		cout << query(l,r) << endl;
	}
	
	return 0;
}
```
有错请指出。参考太戈编程的课件，侵删。