---
layout: post
title: algocon-practice-record-1
postTitle: 算法竞赛做题记录 - 2020年第10周
categories: [Algorithms Competition]
keywords: Algorithms Competition
published: true
mathjax: true
typora-root-url: ..
---

## Problems

###  【CCPC2020 秦皇岛】Gym 102769J - Jewel Splitting

[题目链接](https://vjudge.net/problem/Gym-102769J)

一个长度为$n\ (n\le 3\times 10^5)$的小写字母串，切$\left\lfloor\frac{n}{d}\right\rfloor$刀，得到$\left\lfloor\frac{n}{d}\right\rfloor$个长度为$d$的子串（若$d$不整除$n$，还需要剩下连续的一段$n\bmod d$长的）。现在把长度为$d$的这$\left\lfloor\frac{n}{d}\right\rfloor$段按任意顺序排成一个宽度$d$的矩形。记宽度为$d$的矩形数为$\text{ans}_d$，求

$$
(\sum_{d=1}^{n} \text{ans}_d) \bmod 998244353
$$

#### 做法

哈希 + 动态维护。

对于每个$d$分别计算贡献。

对于单个子串集合所形成矩阵数，只需做字符串哈希和离散化即可。又注意到两个事实：

- 由于剩余段需要是连续的，所以剩余段只有$\left\lceil\frac{n}{d}\right\rceil$个可能位置，即$\left\lceil\frac{n}{d}\right\rceil$种切分方案。因此实际上只会形成$\left\lceil\frac{n}{d}\right\rceil$个字符串集合，且相邻两个集合之间只有一个子串不同
- 如果两种切分方案形成的子串集合不同，那么这两个方案所形成的矩阵也一定互不相同；反之则一定完全相同

因此，我们还需要再做集合哈希，对哈希值相同的集合，只计算一次贡献。

接下来考虑时间复杂度。如果我们能保证计算$\text{ans}_d$的复杂度为$O(\left\lceil\frac{n}{d}\right\rceil)$，那么总的时间复杂度就是$O(nH_n)=O(n\log{n})$，其中$H_n$是调和级数的第$n$个部分和。换句话说，我们需要保证计算$\text{ans}_d$时，只能线性地扫描子串，至多再加一个离散化的排序。这就需要预处理所有可能需要的$d$子串的哈希值及其对应的集合，然后线性地扫描$\left\lceil\frac{n}{d}\right\rceil$个位置，并动态维护集合的哈希值和答案即可。

本题哈希卡自然溢出。

```c++
#include <cstdio>
#include <cstring>
#include <algorithm>
#include <cstdlib>
using namespace std;

#define rep(i, a, b) for (int i = a; i <= b; i++)
#define dep(i, a, b) for (int i = a; i >= b; i--)
#define fill(a, x) memset(a, x, sizeof(a))
#define pb push_back
#define mp make_pair

typedef unsigned long long uLL;
typedef pair<int, int> Pii;
typedef pair<uLL, uLL> Puu;

const int maxN = 600000, N = maxN + 5;
const uLL mod = 998244353ULL;
const uLL mod1 = 19260817ULL, base1 = 661ULL;
const uLL mod2 = 20000623ULL, base2 = 659ULL;
const uLL BASE = 1013ULL;//, CNTB = 109ULL;

int T, len, sub_cnt[N], come_hid[N], leave_hid[N];
uLL hsh1[N], hsh2[N], pw_BASE[N], pw1[N], pw2[N], fac[N], fac_inv[N];
Puu set_hsh_res[N];
pair<Puu, Pii> sub_hsh_tmp[N];
pair<int, Pii> sub_hsh[N];
char str[N];

void do_hash() {
    hsh1[0] = 0ULL;
    hsh2[0] = 0ULL;
    rep(i, 1, len) hsh1[i] = (hsh1[i - 1] * base1 + 1ULL * str[i]) % mod1;
    rep(i, 1, len) hsh2[i] = (hsh2[i - 1] * base2 + 1ULL * str[i]) % mod2;
}

Puu get_hash(int l, int r) {
    uLL val1 = (hsh1[r] - hsh1[l - 1] * pw1[r - l + 1] % mod1 + mod1) % mod1;
    uLL val2 = (hsh2[r] - hsh2[l - 1] * pw2[r - l + 1] % mod2 + mod2) % mod2;
    return mp(val1, val2);
}

uLL qpow(uLL a, uLL m, uLL mod) {
    uLL ret = 1ULL, tmp = a % mod;
    while (m > 0) {
        if (m & 1ULL) 
            ret = mod != 0 ? ret * tmp % mod : ret * tmp;
        tmp = mod != 0 ? tmp * tmp % mod : tmp * tmp;
        m >>= 1;
    }
    return ret;
}

uLL split(int d) {
    int m = len / d;
    int rest = len - m * d;
    int sub_tot = 0;
    rep(i, 1, m) {
        int l = (i - 1) * d + 1, r = l + d - 1;
        sub_hsh_tmp[++sub_tot] = mp(get_hash(l, r), mp(i, m + 1));
        sub_hsh_tmp[++sub_tot] = mp(get_hash(l + rest, r + rest), mp(0, i));
    }
    sort(sub_hsh_tmp + 1, sub_hsh_tmp + sub_tot + 1);
    int sub_val = 0, t = 1;
    while (t <= sub_tot) {
        sub_val++;
        sub_hsh[t] = mp(sub_val, sub_hsh_tmp[t].second);
        while (t < sub_tot && sub_hsh_tmp[t + 1].first == sub_hsh_tmp[t].first) {
            t++;
            sub_hsh[t] = mp(sub_val, sub_hsh_tmp[t].second);
        }
        t++;
    }
    rep(i, 1, sub_tot) {
        pair<uLL, Pii> cur = sub_hsh[i];
        if (cur.second.first != 0) 
            come_hid[cur.second.first] = cur.first;
        if (cur.second.second != m + 1)
            leave_hid[cur.second.second] = cur.first;
    }
    
    uLL res = fac[m];
    rep(i, 1, m + 1) set_hsh_res[i] = mp(0, 0);
    rep(i, 1, sub_val) sub_cnt[i] = 0;
    rep(i, 1, sub_tot) {
        if (sub_hsh[i].second.first == 0) {
            int hsh_val = sub_hsh[i].first;
            sub_cnt[hsh_val]++;
            set_hsh_res[0].first += pw_BASE[hsh_val];
        }
    }
    rep(i, 1, sub_val) res = res * fac_inv[sub_cnt[i]] % mod;
    set_hsh_res[0].second = res;

    rep(i, 1, m) {
        int cur_come = come_hid[i];
        int cur_leave = leave_hid[i];
        res = res * fac[sub_cnt[cur_come]] % mod;
        res = res * fac[sub_cnt[cur_leave]] % mod;
        sub_cnt[cur_come]++;
        sub_cnt[cur_leave]--;
        res = res * fac_inv[sub_cnt[cur_come]] % mod;
        res = res * fac_inv[sub_cnt[cur_leave]] % mod;
        set_hsh_res[i] = mp(set_hsh_res[i - 1].first + pw_BASE[cur_come] - pw_BASE[cur_leave], res);
    }

    uLL ret = 0ULL;
    sort(set_hsh_res, set_hsh_res + m + 1);
    int tot = unique(set_hsh_res, set_hsh_res + m + 1) - (set_hsh_res);
    rep(i, 0, tot - 1) ret = (ret + set_hsh_res[i].second) % mod;
    return ret;
}

int main()
{
    #ifndef ONLINE_JUDGE
        freopen("j.in", "r", stdin);
    #endif

    pw_BASE[0] = pw1[0] = pw2[0] = fac[0] = 1ULL;
    rep(i, 1, maxN) {
        pw_BASE[i] = pw_BASE[i - 1] * BASE;
        pw1[i] = pw1[i - 1] * base1 % mod1;
        pw2[i] = pw2[i - 1] * base2 % mod2;
        fac[i] = fac[i - 1] * (1ULL * i) % mod;
    }
    fac_inv[maxN] = qpow(fac[maxN], mod - 2ULL, mod);
    dep(i, maxN - 1, 0)
        fac_inv[i] = (fac_inv[i + 1] * (1ULL * (i + 1))) % mod;

    int kase = 0;
    scanf("%d", &T);
    while (T--) {
        scanf("%s", str + 1);
        len = strlen(str + 1);
        rep(i, 1, len) str[i] -= ('a' - 1);
        do_hash();

        uLL ans = 0;
        rep(i, 1, len) {
            uLL s = split(i);
            ans = (ans + s) % mod;
        }
        printf("Case #%d: %llu\n", ++kase, ans);
    }
    return 0;
}

```

### 【CCPC2020 秦皇岛】 Gym 102770H - Huge Clouds

[题目链接](https://vjudge.net/problem/Gym-102770H)

#### 做法

计算几何题。题解[见此](http://armeriaw.github.io/2020/10/25/gym-102770h/)。

### 【CCPC2020 秦皇岛】 Gym 102769H - Holy Sequence

[题目链接](https://vjudge.net/problem/Gym-102769H)

序列$\left\{a_1,a_2,\dots,a_n\right\}$合法当且仅当$1\le a_i\le n$且$\max\left\{a_1,a_2,\dots,a_i\right\}−\max\left\{a_1,a_2,\dots,a_{i-1}\right\}\le 1$，$a_1=1$。记长度为$n$的所有合法序列的集合为$S_n$。给定$n\ (n \le 3000)$，要求

$$
\sum_{t=1}^{n}\sum_{p \in S_{n}} \operatorname{cnt}(p, t)^{2},
$$

其中$\operatorname{cnt}(p, t)$表示数字$t$在序列$p$中出现的次数。答案要模上输入值。

#### 做法

不妨考虑**数字$t$第一次出现位置为$x$**的那些序列所贡献的答案，不妨设之为$h[t][x]$。再设$f[i][j]$为长度为$i$、最大值为$j$的合法序列数。在$x$之前的$x-1$个位置上（即前缀$\left\{a_1,\dots,a_{x-1}\right\}$），合法序列数就是$f[x-1][t-1]$。注意前$x-1$项是不能填$t$的，只有后面$n-x + 1$项（即后缀$\left\{a_x,\dots,a_n\right\}$）能填$t$。前后两部分是独立的，所以根据乘法原理，$h[t][x]$就等于后缀$x$贡献的答案乘上$f[x-1][t-1]$。

那么，后缀$x$的答案应该如何计算呢？这里平方的处理思想非常巧妙和重要：做组合意义的转化。$\operatorname{cnt}(p, t)^{2}$就相当于在$p$的所有$t$的出现中，选择两个的方案数（可以重复选同一个）。例如，设$p=\left\{1, 1, 2, 3, 2, 4, 2\right\}$，$t=2$，则从$p$的所有$3$次$t$的出现中选出$2$个的方案数就是$\operatorname{cnt}(p, t)^{2}=9$。

有了这个组合意义，就可以DP后缀$x$的答案了。设$d[i][j][0/1/2]$为长度为$i$、最大值为$j$、已经选择指定数字$0/1/2$次的方案数（因为指定数字一定在$[1,j]$范围内，且都是等价的，所以不必指明是哪一个）。之所以要这么设计状态数组，是因为这样一来$d$的计算与所选择数字$t$就是无关的。换句话说，我们可以事先预处理$d$数组；对于每个数字$t$，我们都能重复利用它。

$d$的转移也是十分自然的，分类讨论即可。考虑如下的记忆化搜索：

```python
# 对t=0，考虑第i位的填法和选法
d[i][j][0] -> d[i + 1][j + 1][0]        # 填j+1
d[i][j][0] -> d[i + 1][j][0] * (j - 1)  # 填1~j中的一个数（非指定数）
d[i][j][0] -> d[i + 1][j][0]            # 填指定数，但不选
d[i][j][0] -> d[i + 1][j][1] * 2        # 填指定数，只选一次
d[i][j][0] -> d[i + 1][j][2]            # 填指定数，选两次
```

时间复杂度就是$O(n^2)$的。如下的代码就是用这个思路写的：

```c++
#include <cstdio>
#include <cstring>
#include <algorithm>
using namespace std;

#define rep(i, a, b) for (int i = a; i <= b; i++)
#define dep(i, a, b) for (int i = a; i >= b; i--)
#define fill(a, x) memset(a, x, sizeof(a))
#define pb push_back
#define mp make_pair

typedef int ll;

const int N = 3000 + 5;

ll d[N][N][3], f[N][N], res[N], mod;
int T, n;

inline void mod_add(ll &res, ll x) {
    while (res >= mod) res -= mod;
    while (x >= mod) x -= mod;
    res += x;
    while (res >= mod) res -= mod;
}

// 还需要决策pos个位置, 已填的最大数是max_num, 需要选指定数k=0/1/2次
inline ll dp(int pos, int max_num, int k) {
    ll &cur = d[pos][max_num][k];
    if (cur != -1) return cur;
    if (pos <= 0) {
        if (k == 0) return cur = 1;
        return cur = 0;
    }
    cur = 0;
    if (k == 2) {
        mod_add(cur, dp(pos - 1, max_num + 1, 2));
        mod_add(cur, (1LL * dp(pos - 1, max_num, 2) * (max_num - 1)) % mod);
        mod_add(cur, dp(pos - 1, max_num, 2));
        mod_add(cur, dp(pos - 1, max_num, 1) * 2);
        mod_add(cur, dp(pos - 1, max_num, 0));
    }
    else if (k == 1) {
        mod_add(cur, dp(pos - 1, max_num + 1, 1));
        mod_add(cur, (1LL * dp(pos - 1, max_num, 1) * (max_num - 1)) % mod);
        mod_add(cur, dp(pos - 1, max_num, 1));
        mod_add(cur, dp(pos - 1, max_num, 0));
    }
    else {  // k = 0
        mod_add(cur, dp(pos - 1, max_num + 1, 0));
        mod_add(cur, (1LL * dp(pos - 1, max_num, 0) * max_num) % mod);
    }
    return cur;
}

int main()
{
    #ifndef ONLINE_JUDGE
        freopen("h.in", "r", stdin);
    #endif

    int kase = 0;
    scanf("%d", &T);
    while (T--) {
        scanf("%d%d", &n, &mod);
        rep(i, 0, n) {
            res[i] = 0;
            rep(j, 0, n) {
                f[i][j] = 0;
                d[i][j][0] = d[i][j][1] = d[i][j][2] = -1;
            }
        }
        rep(i, 1, n) dp(n, i, 2); 
        f[0][0] = 1;
        rep(i, 1, n) rep(j, 1, i) 
            mod_add(f[i][j], (1LL * f[i - 1][j] * j + f[i - 1][j - 1]) % mod);
        rep(i, 1, n)  // num
            rep(j, i, n) {  // start pos of 2nd part
                ll tmp = d[n - j][i][0];
                mod_add(tmp, d[n - j][i][1] * 2);
                mod_add(tmp, d[n - j][i][2]);
                mod_add(res[i], (1LL * f[j - 1][i - 1] * tmp) % mod);
        }
        printf("Case #%d:\n", ++kase);
        rep(i, 1, n - 1) printf("%d ", res[i]);
        printf("%d\n", res[n]);
    }
    return 0;
}
```

但由于记忆化搜索太慢，尽管我各种卡常，这代码最后还是TLE。（也许把递归改成手写栈能过，没试。）

还有另一种更高效的递推做法，但比较难想。我们先不看式子里的平方，假设要求的是

$$
\sum_{t=1}^{n}\sum_{p \in S_{n}} \operatorname{cnt}(p, t).
$$

仍然考虑数字$t$第一次出现位置为$x$的序列数。设合法前缀数为$f[i][j]$（长度为$i$，最大值为$j$），合法后缀数为$g[i][j]$（长度为$i$，以数字$j$开头），则合法序列数为$f[x-1][t-1]\times g[n-x+1][t]$。

接下来考虑怎么利用$g$数组计算我们想要的答案。还是分类讨论：

- 选两次$a[x]$。对答案贡献就是$g[n-x+1][t]$
- 选一次$a[x]$，选一次后面的$t$。这需要保证后面还有至少一个$t$。其实可以这么想：我们空出一个位置出来，让一个$t$随意插入。这样，方案数就是$2\times g[n-x][t]\times (n - x)$。

- 选两个相同的$t$，但不是$a[x]$。思考方式同上，方案数是$g[n-x][t]\times (n - x)$。
- 选两个不同的$t$，但都不是$a[x]$。这时需要空出两个位置，方案数为$g[n-x-1][t]\times(n-x-1)\times(n-x)$。

最后一个小问题是$g[i][j]$如何计算。考虑从后往前填数，第$i$步填上的第$1$位根据定义就是$j$：

- 如果第$2$位填$[1,j]$之间的数，那么方案数就是$g[i-1][j]\times j$（这是因为我们要的是第$3$位及之后的变化。第$1$位已经填上$j$了，所以无论第$2$位是$[1,j]$的哪一个数，这个变化数都是相同的$g[i-1][j]$）。
- 如果第$2$位填$j+1$，方案数就是$g[i - 1][j + 1]$。

因此有递推式$g[i][j] = g[i-1][j]\times j + g[i-1][j+1]$。

代码如下：

```c++
#include <bits/stdc++.h>
using namespace std;

#define rep(i, a, b) for (int i = a; i <= b; i++)
#define dep(i, a, b) for (int i = a; i >= b; i--)
#define vep(i, v) for (int i = 0; i < (int)v.size(); i++)
#define fill(a, x) memset(a, x, sizeof(a))
#define mp make_pair
#define pb push_back

typedef long long ll;

const int N = 3000 + 5;

int T, n;
ll mod;
ll f[N][N], g[N][N], ans[N];

int main()
{
    int kase = 0;
    scanf("%d", &T);
    while (T--) {
        scanf("%d%lld", &n, &mod);
        f[0][0] = 1;
        rep(i, 1, n) rep(j, 1, i) 
            f[i][j] = (f[i - 1][j] * j + f[i - 1][j - 1]) % mod;
        rep(i, 1, n) g[1][i] = 1;
        rep(i, 2, n) rep(j, 1, n - i + 1)
            g[i][j] = (g[i - 1][j] * j + g[i - 1][j + 1]) % mod;
        rep(i, 1, n) {
            ans[i] = 0;
            rep(j, i, n) {
                ll tmp = g[n - j + 1][i];
                if (j < n) tmp = (tmp + 3 * g[n - j][i] * (n - j)) % mod;
                if (j < n - 1) tmp = (tmp + g[n - j - 1][i] * (n - j - 1) % mod * (n - j)) % mod;
                ans[i] = (ans[i] + f[j - 1][i - 1] * tmp) % mod;
            }
        }
        printf("Case #%d:\n", ++kase);
        rep(i, 1, n - 1) printf("%lld ", ans[i]);
        printf("%lld\n", ans[n]);
    }
    return 0;
}
```



### 【ICPC2019 香港】Gym 102452C - Constructing Ranches

[题目链接](https://vjudge.net/problem/Gym-102452C)

给出$n\left(1 \leq n \leq 2 \times 10^{5}\right)$个点的树，点上有权。问有多少点对$(u,v)$，使得$u$到$v$的路径上的那些点权能够构成简单多边形。

#### 做法

点分治好题。

简单多边形（无论凸还是凹）满足三角形不等式

$$
\sum_{i=1}^{n} a_{i}>2 \max _{i=1}^{n}\left\{a_{i}\right\}.
$$

因此点分治求出重心$c$之后，子树中的答案递归计算；现在集中精力求经过$c$的路径数（即点对数）。我们需要维护两个东西：$s_u$和$m_u$分别表示$c$到$u$路径上的点权和以及最大点权。枚举路径的一端$u$，则需要统计的路径数就是满足

$$
s_{u}+s_{v}-a_{c}>2 \max \left\{m_{u}, m_{v}\right\}
$$

的结点$v$的数量。

下面的操作是关键。首先，式子里的$\max$很不友好，我们不希望做分类讨论。为此，我们按照$m$从小到大的顺序依次固定每个点$u$，且只统计$m_v \le m_u$的结点$v$。这样，就变成了统计$s_v > 2m_u - s_u + a_c$的$v$的数量。我们只要事先把$s$数组排个序，就可以直接在$s$数组上二分了。但是，如何保证只统计$m_v \le m_u$的$v$呢？别忘了我们是按$m$升序依次固定每个点的，所以$m_v \le m_u$的$v$就是先前已经固定过的点。因此，我们在每次计算完固定点$u$的答案之后，找到$u$在排序好的$s$数组中的位置$q$，并在$q$处做一个标记。（如何找到？预处理映射。）这样，下一个固定点$u'$的答案，就是在$s$数组上二分得到的位置$p$后面的标记总数。为了快速统计这个数量，可以用树状数组维护之。

实现细节上，由于树状数组一般都是用来统计数组中$\le$某个位置的元素和，为了写着顺溜，可以按$s$倒序排序，也可以拿总的$v$数量减去不满足条件（换句话说就是满足$s_v \le 2m_u - s_u + a_c$）的$v$的数量。下面的代码实现使用的是后一种。

```c++
#include <bits/stdc++.h>
using namespace std;

#define rep(i, a, b) for (int i = a; i <= b; i++)
#define dep(i, a, b) for (int i = a; i >= b; i--)
#define vep(i, v) for (int i = 0; i < (int)v.size(); i++)
#define fill(a, x) memset(a, x, sizeof(a))
#define mp make_pair
#define pb push_back

typedef long long ll;
typedef unsigned long long ull;
typedef pair<int, int> pii;
typedef pair<ll, int> pli;
typedef pair<ll, ll> pll;

const int N = 200000 + 5;
const ll INF = 1e18;

int T, n, u, v, sub_size[N];
ll ans, a[N];
bool centroid[N];
vector<int> G[N];

inline int calc_subtree(int x, int dad) {
    int sz = 1;
    for(int y: G[x]) {
        if (y == dad || centroid[y]) continue;
        sz += calc_subtree(y, x);
    }
    return sub_size[x] = sz;
}

inline pii search_centriod(int x, int dad, int tot) {
    pii res = mp(N, -1);
    int s = 1, m = 0;
    for (int y: G[x]) {
        if (y == dad || centroid[y]) continue;
        res = min(res, search_centriod(y, x, tot));
        m = max(m, sub_size[y]);
        s += sub_size[y];   
    }
    m = max(m, tot - s);
    res = min(res, mp(m, x));
    return res;
}

inline void calc_len(int x, int dad, ll len, ll mx, 
                    vector<pli> &sub_sum, vector<pli> &sub_max) {
    sub_sum.pb(mp(len, x));
    sub_max.pb(mp(mx, x));
    for (int y: G[x]) {
        if (y == dad || centroid[y]) continue;
        calc_len(y, x, len + a[y], max(mx, a[y]), sub_sum, sub_max);
    }
}

int ord_sum[N], tree[N];
inline int lowbit(int x) { return x & -x; }
inline void update(int x, int v, int upper) {
    for (int i = x; i <= upper; i += lowbit(i)) tree[i] += v;
}
inline int query(int x) {
    int ret = 0;
    for (int i = x; i; i -= lowbit(i)) ret += tree[i];
    return ret;
}

inline ll count_pairs(vector<pli> &sub_sum, vector<pli> &sub_max, ll ax) {
    ll ret = 0;
    int sz = sub_max.size();
    sort(sub_sum.begin(), sub_sum.end());
    sort(sub_max.begin(), sub_max.end());
    vep(i, sub_sum) ord_sum[sub_sum[i].second] = i;
    rep(i, 1, sz) tree[i] = 0;
    rep(i, 0, sz - 1) {
        int u = sub_max[i].second;
        ll mu = sub_max[i].first, su = sub_sum[ord_sum[u]].first;

        if (i > 0) {
            ll limit = (mu << 1) + ax - su;
            int l = -1, r = sz;
            while (l + 1 < r) {
                int mid = (l + r) >> 1;
                if (sub_sum[mid].first > limit) r = mid; else l = mid;
            }
            ll inc = i - query(l + 1);
            ret += inc;
        } 
        update(ord_sum[u] + 1, 1, sz);
    }
    return ret;
}

inline void divide_and_solve(int x) {
    calc_subtree(x, -1);
    int s = search_centriod(x, -1, sub_size[x]).second;
    centroid[s] = true;
    vep(i, G[s]) {
        int y = G[s][i];
        if (centroid[y]) continue;
        divide_and_solve(y);
    }
    vector<pli> sub_sum;
    vector<pli> sub_max;
    sub_sum.pb(mp(a[s], s));
    sub_max.pb(mp(a[s], s));
    vep(i, G[s]) {
        int y = G[s][i];
        if (centroid[y]) continue;
        vector<pli> tmp_sub_sum;
        vector<pli> tmp_sub_max;
        calc_len(y, s, a[s] + a[y], max(a[s], a[y]), tmp_sub_sum, tmp_sub_max);
        sub_sum.insert(sub_sum.end(), tmp_sub_sum.begin(), tmp_sub_sum.end());
        sub_max.insert(sub_max.end(), tmp_sub_max.begin(), tmp_sub_max.end());
        ans -= count_pairs(tmp_sub_sum, tmp_sub_max, a[s]);
    }
    ans += count_pairs(sub_sum, sub_max, a[s]);
    centroid[s] = false;
}

int main()
{
    #ifndef ONLINE_JUDGE
        freopen("c.in", "r", stdin);
    #endif

    scanf("%d", &T);
    while (T--) {
        scanf("%d", &n);
        rep(i, 1, n) G[i].clear(), scanf("%lld", &a[i]);
        rep(i, 1, n - 1) {
            scanf("%d%d", &u, &v);
            G[u].pb(v), G[v].pb(u);
        }
        ans = 0;
        divide_and_solve(1);
        printf("%lld\n", ans);
    }
    return 0;
}
```

### HDU 4009 - Transfer water

[题目链接](https://vjudge.net/problem/HDU-4009)

#### 做法

最小树形图裸题。建立虚根后直接套朱-刘算法。板子和注释见下。

```c++
#include <bits/stdc++.h>
using namespace std;

#define rep(i, a, b) for (int i = a; i <= b; i++)
#define dep(i, a, b) for (int i = a; i >= b; i--)
#define vep(i, v) for (int i = 0; i < v.size(); i++)
#define fill(a, x) memset(a, x, sizeof(a))
#define mp make_pair
#define pb push_back

const int N = 1000 + 5;
const int INF = 1e9;

int n, x, y, z, s, t;
int fa[N], a[N], b[N], c[N], que[N], G[N][N];
bool canb[N][N], used[N], pass[N], more;

inline void subtract(int id, int &res) {
    int head = 1, tail = 0;
    while (id != 0 && !pass[id]) {
        que[++tail] = id;
        pass[id] = true;
        id = fa[id];
    }
    while (head <= tail && que[head] != id) head++;
    // 无圈，直接返回
    if (head == tail + 1) return;

    // 有圈，需要继续迭代
    // 此时que[head] == id为圈中第一个被访问到的点，也是圈的代表元
    // que[head]到que[tail]就是圈上的所有点（无重复）
    more = true;
    rep(i, head, tail) {
        int cur = que[i];
        res += G[fa[cur]][cur];
        if (i == head) continue;
        // 除了代表元id，其他点都缩掉
        used[cur] = true;
        // 处理缩点的所有出边
        rep(j, 1, n) if (!used[j])
           G[id][j] = min(G[id][j], G[cur][j]);
    }
    // 处理缩点的所有入边
    rep(i, 1, n) if (!used[i] && i != id) {
        rep(j, head, tail) {
            int k = que[j];
            G[i][id] = min(G[i][id], G[i][k] - G[fa[k]][k]);
        }
    }
}

inline int zhuliu(int rt) {
    int res = 0;
    // used[i]表示点i是否被缩了
    fill(used, false);
    more = true;
    while (more) {
        more = false;
        fill(fa, 0);
        rep(i, 1, n) if (!used[i] && i != rt) {
            int k = 0;
            rep(j, 1, n) if (!used[j] && i != j) {
                if (k == 0 || G[j][i] < G[k][i]) k = j;
            }
            fa[i] = k;
        }
        // pass[i]表示点i是否在当前这轮迭代被访问过
        fill(pass, false);
        rep(i, 1, n) if (!used[i] && !pass[i] && i != rt) 
            subtract(i, res);
    }
    rep(i, 1, n) if (!used[i] && i != rt) res += G[fa[i]][i];
    return res;
}

int main()
{
    #ifndef ONLINE_JUDGE
        freopen("in.txt", "r", stdin);
    #endif

    while (~scanf("%d%d%d%d", &n, &x, &y, &z) && n) {
        n++;
        fill(canb, false);
        rep(i, 2, n) scanf("%d%d%d", &a[i], &b[i], &c[i]);
        rep(i, 2, n) {
            scanf("%d", &s);
            rep(j, 1, s) {
                scanf("%d", &t);
                canb[i][t + 1] = true;
            }
        }
        rep(i, 2, n) rep(j, 2, n) {
            if (i == j || !canb[i][j]) {
                G[i][j] = INF;
                continue;
            }
            G[i][j] = (abs(a[i] - a[j]) + abs(b[i] - b[j]) + abs(c[i] - c[j])) * y + (c[i] < c[j]) * z;
        }
        rep(i, 2, n) G[1][i] = c[i] * x;
        printf("%d\n", zhuliu(1));
    }

}
```

### 【ICPC2019 银川】计蒜客 42388 - Delivery Route

[题目链接](https://vjudge.net/problem/计蒜客-42388/origin)

给定一张$n\ (1 \leq n \leq 25000)$个点$m\ (1\le m\le 10^5)$条边混合图，其中负权边只会出现在有向边上。此外，对于任一有向边$(u,v)$，从$v$出发是不能返回$u$的。求满足上述性质的图的单源最短路。

#### 做法

这种题毫无疑问会卡掉SPFA。

注意这个关键性质，「对有向边$(u,v)$，从$v$出发不能返回$u$」。显然可以推出，$u$和$v$不在一个SCC中。进一步地，边$(u,v)$也不在SCC中，它是连接两个SCC的边。因为所有有向边都不在SCC中，所以构成SCC的边只可能是无向边，而无向图的SCC其实就是连通分量。因此，整张混合图的形态就是若干个无向正权的连通分量，由若干条可能为负权的有向边连接。

这样，我们只需对无向连通分量做Dijkstra，有向边做拓扑即可。注意，这里拓扑图的结点是缩点后的连通分量，因此正确的拓扑排序流程是，等这个连通分量的所有入度都删掉之后，把这整个连通分量加入队列。

下面的代码写得有点烦琐，其实可以在Dijkstra中顺便处理有向边、更新入度。

```c++
#include <cstdio>
#include <cstring>
#include <algorithm>
#include <cstdlib>
#include <vector>
#include <queue>
using namespace std;

#define rep(i, a, b) for (int i = a; i <= b; i++)
#define dep(i, a, b) for (int i = a; i >= b; i--)
#define vep(i, v) for (int i = 0; i < (int)v.size(); i++)
#define fill(a, x) memset(a, x, sizeof(a))
#define pb push_back
#define mp make_pair

typedef unsigned long long uLL;
typedef pair<int, int> Pii;
typedef pair<uLL, uLL> Puu;

const int N = 100000 + 5;

int n, x, y, s, u, v, w, top, cc_cnt, dfs_clock;
int bel[N], dis[N], ind[N];
bool vis[N];
vector<int> cc[N];
vector<Pii> Gd[N], Gs[N];
queue<int> que;

void dfs1(int x) {
    vis[x] = true;
    bel[x] = cc_cnt;
    cc[cc_cnt].pb(x);
    vep(i, Gd[x]) {
        int y = Gd[x][i].first;
        if (!vis[y]) dfs1(y);
    }
}

void dfs2(int x) {
    vis[x] = true;
    vep(i, Gd[x]) {
        int y = Gd[x][i].first;
        if (!vis[y]) dfs2(y);
    }
    vep(i, Gs[x]) {
        int y = Gs[x][i].first;
        if (!vis[y]) dfs2(y);
    }
}

priority_queue<Pii, vector<Pii>, greater<Pii> > pq;
void dijkstra(int s) {
    while (!pq.empty()) pq.pop();
    int c = bel[s];
    vep(i, cc[c]) {
        int x = cc[c][i];
        if (dis[x] == 0x3f3f3f3f) continue;
        pq.push(mp(dis[x], x));
    }
    while (!pq.empty()) {
        int x = pq.top().second;
        pq.pop();
        if (vis[x]) continue;
        vis[x] = true;
        vep(i, Gd[x]) {
            int y = Gd[x][i].first;
            if (bel[y] != bel[x]) continue;

            int w = Gd[x][i].second;
            if (dis[y] > dis[x] + w) {
                dis[y] = dis[x] + w;
                pq.push(mp(dis[y], y));
            }
        }
    }
}


int main() 
{
    #ifndef ONLINE_JUDGE
        freopen("h.in", "r", stdin);
    #endif

    scanf("%d%d%d%d", &n, &x, &y, &s);
    rep(i, 1, n) Gd[i].clear(), Gs[i].clear(), cc[i].clear();
    rep(i, 1, x) {
        scanf("%d%d%d", &u, &v, &w);
        Gd[u].pb(mp(v, w));
        Gd[v].pb(mp(u, w));
    }

    cc_cnt = 0;
    fill(vis, false);
    rep(i, 1, n) if (!vis[i]) {
        cc_cnt++;
        dfs1(i);
    }

    rep(i, 1, y) {
        scanf("%d%d%d", &u, &v, &w);
        Gs[u].pb(mp(v, w));
    }
    fill(ind, 0);
    fill(vis, false);
    dfs2(s);
    rep(x, 1, n) {
        if (!vis[x]) continue;
        vep(i, Gs[x]) 
            ind[bel[Gs[x][i].first]]++;
    }

    fill(dis, 0x3f3f3f3f);
    fill(vis, false);
    while (!que.empty()) que.pop();
    que.push(s);
    dis[s] = 0;
    while (!que.empty()) {
        int x = que.front();
        que.pop();
        dijkstra(x);

        int c = bel[x];
        vep(i, cc[c]) {
            int a = cc[c][i];
            vep(j, Gs[a]) {
                int b = Gs[a][j].first;
                int w = Gs[a][j].second;
                ind[bel[b]]--;
                if (ind[bel[b]] == 0) que.push(b);
                if (dis[b] > dis[a] + w) {
                    dis[b] = dis[a] + w;
                }
            }
        }
    }
    rep(i, 1, n) {
        if (dis[i] == 0x3f3f3f3f) {
            puts("NO PATH");
        } else {
            printf("%d\n", dis[i]);
        }
    }

    return 0;
}
```

## Tips

### 求逆元表

求$[1,n]$逆元表

```C++
typedef  long long ll;
const int N = 1e5 + 5;
int inv[N];
 
void inverse(int n, int p) {
    inv[1] = 1;
    for (int i=2; i<=n; ++i) {
        inv[i] = (ll) (p - p / i) * inv[p%i] % p;
    }
}
```

求$1!, 2!, \dots, n!$的逆元表
```C++

uLL qpow(uLL a, uLL m, uLL mod) {
    uLL ret = 1ULL, tmp = a % mod;
    while (m > 0) {
        if (m & 1ULL) 
            ret = mod != 0 ? ret * tmp % mod : ret * tmp;
        tmp = mod != 0 ? tmp * tmp % mod : tmp * tmp;
        m >>= 1;
    }
    return ret;
}


// fac[maxN]为最大值maxN的阶乘
fac_inv[maxN] = qpow(fac[maxN], mod - 2ULL, mod);
dep(i, maxN - 1, 0)
    fac_inv[i] = (fac_inv[i + 1] * (1ULL * (i + 1))) % mod;
```

### STL

用`unique`之前要先`sort`！

`vector`、`queue`、`set`等容器的`size()`时间复杂度都是$O(1)$

### 计算几何

- 简单多边形（simple polygon）包括凸多边形和凹多边形。简单多边形，无论凸还是凹，都满足三角形不等式

### 赌徒破产问题

赌徒甲有资本$a$元，赌徒乙有资本$b$元，两人进行赌博，每赌一局输者给赢者$1$元，没有和局，直赌至两人中有一人输光为止。设在每一局中，甲获胜的概率为$p$，乙获胜的概率为$q=1-p$，求甲先输光的概率。

设$c=a+b$，$r=q/p$。则

<img src="https://pic002.cnblogs.com/images/2012/382146/2012030810002368.png" alt="img" style="zoom:67%;" />

这里面的高阶乘方计算要注意精度。