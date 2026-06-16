---
title: LCA算法
date: 2026-06-16 19:07:56
updated: 2026-06-16 19:07:56
categories:
    - 算法学习
tags: [LCA, DFS, 倍增, 离线算法, ST表, RMQ问题, 重链剖分]
---
*拖延了好久，终于找时间写LCA算法整合了*

{% note info %}
## 什么是LCA？
**LCA**，即Lowest Common Ancestor，最近公共祖先，指在一棵树上两个点的公共祖先中深度最大的那个祖先。
在这里，我会分别介绍**朴素法**，**倍增法**，**Tarjan**离线算法，**欧拉序**+RMQ，**DFN序**+RMQ，**重链剖分(HLD)**。

**建议结合代码观看**

**最常用的是倍增法**
**RMQ**(Range Minimum/Maximum Query)，这里用ST表实现。
$\text{*}$这里重链剖分得到的DFN序在求LCA时并不会用到，实际上重链剖分更重要的作用是可以结合线段树将树上问题转化为区间问题。

如果以后有可能的话，我会去学习动态树求LCA（Link-Cut Tree）
{% endnote %}

## 朴素法
{% note primary %}
**维护信息**：
节点的**父亲**$F_x$以及**深度**$H_x$。

**算法步骤**：
先通过一遍DFS遍历树，维护节点父亲和节点深度。

对于树上两点u和v，比较两者深度，如果两者深度不同，则不断升高深度更大的点，直至两者深度相同。
此时如果两个节点为同一节点的话，就已经找到了LCA。
否则，让两个节点同步上升，直到两者相等。相等时的节点就是所求的LCA。

**显然该算法复杂度为O(H)**
{% endnote %}

**这里就不放代码了，直接进入倍增法**

## 倍增法
{% note primary %}
**倍增法**由朴素法优化而来，观察朴素法的步骤，每次节点只能上升1的高度，过于低效，倍增法就是来解决这个问题的。

**维护信息**：
节点向上数的第$2^k$个**祖先**$F_{x,k}$，以及节点**深度**$H_x$。

**算法步骤**：
先通过一遍DFS遍历树，维护节点的$2^k$级祖先和节点深度。
其中，$2^k$级祖先通过如下公式
$$
\begin{cases}
    F_{x,0}=F_x\\
    F_{x,k}=F_{F_{x,k-1},k-1}
\end{cases}
$$

描述的就是：x的第$2^k$个祖先就是x的第$2^{k-1}$个祖先的第$2^{k-1}$个祖先。

寻找LCA的方法和倍增法类似，但有些不同。
在最初的上升至同一深度阶段中：记录两点的深度差后，采用和**快速幂**一样的方法。例如数字13的二进制表示为1101，可以拆成8+4+1，只需要向上跳2的0，2，3次幂即可。
随后判断节点是否相同，若相同直接返回当前节点。
若不相同，则一起向上跳，这里的操作为，2的高次幂开始尝试，如果两者对应祖先节点不相同，则分别跳到该祖先节点。直到$2^0=1$个祖先，即他们的父亲节点相同。
最终返回他们的父亲节点，就是所求LCA。

**复杂度**分析：
在DFS预处理阶段需要用$\log$来处理祖先节点，因此复杂度为$O(N\log N)$
查询时是倍增，复杂度为$O(\log H)$
{% endnote %}

**Code**
在后续将不会展示sol函数。
```cpp
vector<vector<int>>edge(maxn),f(maxn,vector<int>(21));
vector<int>h(maxn);

void dfs(int x,int fa)
{
    f[x][0]=fa;
    h[x]=h[fa]+1;
    for(int i=1;i<21;i++)
    {
        f[x][i]=f[f[x][i-1]][i-1];
    }
    for(auto v:edge[x])
    {
        if(v==fa)continue;
        dfs(v,x);
    }
}

int LCA(int a,int b)
{
    if(h[a]>h[b])swap(a,b);
    int gap=h[b]-h[a];
    int w=0;
    while(gap)
    {
        if(gap&1)b=f[b][w];
        gap>>=1;
        w++;
    }
    if(a==b)return a;
    for(int i=20;i>=0;i--)
    {
        if(f[a][i]==f[b][i])continue;
        a=f[a][i];
        b=f[b][i];
    }
    return f[a][0];
}

void sol()
{
    int N,M,S;
    cin>>N>>M>>S;
    for(int i=1;i<N;i++)
    {
        int u,v;
        cin>>u>>v;
        edge[u].push_back(v);
        edge[v].push_back(u);
    }
    dfs(S,0);
    while(M--)
    {
        int a,b;
        cin>>a>>b;
        cout<<LCA(a,b)<<'\n';
    }
}
```

## Tarjan离线算法
{% note warning %}
可以作为**离线思想**的入门。
你需要掌握**并查集**的前置知识。
{% endnote %}

{% note primary %}
**维护信息**：
用$vis$维护点在DFS中是否已访问
离线信息：对于第$i$个询问$(u,v)$，在点$u$上记录$v$和$i$，在点$v$上记录$u$和$i$。

**算法步骤**：
用DFS遍历树，每次进入一个节点x，标记$vis_x$为已访问。
（在进入子节点前，需要**处理询问信息**，但这里先不说，在后续解释。）
在遍历完节点x的所有的子节点后的回溯阶段，将x合并到父亲节点中。

在这个合并的过程中，x所构成的子树均会合并到x的父节点fa上，此时如果父节点继续进入其他子节点，假设进入了节点v，处理v的所有**询问信息**，假设(u,v)是一对询问且u在x的子树中，此时u节点已经被访问过，而通过并查集查找u的头节点就是fa，也就是两者的LCA。
以上是在两个子树中的情况，而对于同一条链上求LCA是否符合呢？设(u,v)在DFS遍历的同一条链上，设u深度更低，在递归到v时会处理(u,v)的询问信息，此时u已经被访问过，查询u的头节点，由于u还未进行过合并，头节点指向自己，因此u就是LCA，符合实际。

**处理询问信息**：
处理节点x的所有询问信息(x,v)，如果v已被访问过，则LCA为v的头结点。

**复杂度**分析：
只需要一次DFS，并查集用路径压缩优化，最终复杂度略大于$O(N)$。
{% endnote %}

**Code**
```cpp
int Find(int x)
{
    if(x==F[x])return x;
    return F[x]=Find(F[x]);
}
void merge(int x,int fa)
{
    F[Find(x)]=Find(fa);
}

void Tarjan_DFS(int x,int fa=0)
{
    vis[x]=1;
    for(auto [y,id]:query[x])
    {
        if(!vis[y])continue;
        ans[id]=Find(y);
    }
    for(auto v:edge[x])
    {
        if(v==fa)continue;
        Tarjan_DFS(v,x);
    }
    merge(x,fa);
}
```

## 欧拉序和DFN序+RMQ
{% note primary %}
**欧拉序**：
在DFS遍历树时，在每次进入x节点和回到x节点均记录x，构成的序列为欧拉序。
**DFN序**：
DFS遍历树的顺序，只需要再每一次进入节点时记录即可。

**在欧拉序中寻找LCA**：
记录节点x**第一次**在欧拉序中出现的时间$T_x$。
寻找LCA(u,v)，则在欧拉序的$T_u$到$T_v$之间，深度最小的节点即为所求LCA。
简单证明：
设$T_u\le T_v$，根据欧拉序的性质，在这个区间段内，会记录u构成的子树中的所有节点，以及从u回溯到LCA，从LCA下沉到v节点路径中的所有节点。由于比LCA的子树并未完全遍历，不会回溯到比LCA深度还低的节点，因此深度最低的节点就是LCA。

**在DFN序中寻找LCA**：
记录节点x在DFNF序中出现的时间$T_x$。
寻找LCA(u,v)，由于DFN序不像欧拉序一样连续，存在跳跃性，设$T_u<T_v$，则从u上升到LCA的路径不会被记录，LCA也同样不会被记录，但是在从LCA下沉到v的路径会被记录，**而下沉时一定记录了LCA的一个子节点**，因此我们需要寻找这一段DFN序记录的节点中，深度最低的父亲节点就是LCA。
！！！这里注意，查询的区间为$\left( T_u,T_v\right]$，因为如果u和v在一条链上，那么深度最低节点为LCA的父亲节点。

复杂度：ST表预处理$O(N\log N)$，查询$O(1)$
{% endnote %}

**欧拉序Code**
```cpp
void dfs(int x,int fa)
{
    h[x]=h[fa]+1;
    ft[x]=euler.size();
    euler.push_back(x);
    for(auto v:edge[x])
    {
        if(v==fa)continue;
        dfs(v,x);
        euler.push_back(x);
    }
}

int MIN(int a,int b)
{
    if(h[a]<h[b])return a;
    else return b;
}

void build_ST()
{
    for(int i=0;i<euler.size();i++)ST[i][0]=euler[i];
    lg[1]=0;
    for(int i=2;i<=euler.size();i++)lg[i]=lg[i/2]+1;
    for(int i=1;i<21;i++)
    {
        int len=1<<i;
        for(int j=0;j+len<=euler.size();j++)
        {
            ST[j][i]=MIN(ST[j][i-1],ST[j+(1<<(i-1))][i-1]);
        }
    }
}

int LCA(int a,int b)
{
    int l=ft[a],r=ft[b];
    if(l>r)swap(l,r);
    int k=lg[r-l+1];
    return MIN(ST[l][k],ST[r-(1<<k)+1][k]);
}
```
**DFN序Code**
```cpp
void dfs(int x,int fa)
{
    h[x]=h[fa]+1;
    f[x]=fa;
    ft[x]=dfn.size();
    dfn.push_back(x);
    for(auto v:edge[x])
    {
        if(v==fa)continue;
        dfs(v,x);
    }
}

int MIN(int a,int b)
{
    if(h[f[a]]<h[f[b]])return a;
    else return b;
}

int LCA(int a,int b)
{
    if(a==b)return a;
    int l=ft[a],r=ft[b];
    if(l>r)swap(l,r);
    l++;
    int k=lg[r-l+1];
    int res=MIN(ST[l][k],ST[r-(1<<k)+1][k]);
    return f[res];
}
```

## 重链剖分(HLD)
{% note %}
用它来求LCA有点杀鸡用牛刀了。。。
{% endnote %}

{% note primary %}
定义：
**重孩子**(Heavy Child)：子节点构成的子树中节点最多的孩子。
**重链**(Heavy Chain)：由**重链头**（可能不是重孩子）和随后连续的重孩子构成的链。
**轻链**(Light Chain)：重链之间由轻链相连。

可知树上所有边均属于于重链或轻链。
可知一个树最多$\log N$条重链，因为重链上斗士重孩子，在非常均匀的情况下才能达到$\log N$条重链。

**维护信息**：
每个节点的**父亲**，**子树大小**，**重孩子**，**所在重链的头节点**，**节点深度**。（同时可以维护第二次DFS的DFN序，但是只求LCA是用不到的）

**算法步骤**：
第一个DFS，维护父亲节点，子树大小，重孩子，节点深度。
第二个DFS，维护所在重链的头节点。（在这里维护DFN序）
对于节点(u,v)，如果在同一条重链上，则深度更小的为所求LCA。
如果两者不在同一条重链上，比较两条重链**链头的深度**，不妨设$H_{head_u}<H_{head_v}$。将v跳到其链头的父亲节点（即通过一条轻链进入了另一条重链），随后继续进行该操作，直到两者在同一条重链上，得出结果。

复杂度和倍增法类似，均是$O(\log N)$的查询速度。
{% endnote %}

**Code**
```cpp
vector<vector<int>>edge(maxn);
vector<int>F(maxn),head(maxn),sz(maxn),hs(maxn),h(maxn),dfn;

void dfs1(int x,int fa=0)
{
    F[x]=fa;
    sz[x]=1;
    head[x]=x;
    h[x]=h[fa]+1;
    int HS=0,SZ=0;
    for(auto v:edge[x])
    {
        if(v==fa)continue;
        dfs1(v,x);
        sz[x]+=sz[v];
        if(sz[v]>SZ)
        {
            SZ=sz[v];
            HS=v;
        }
    }
    hs[x]=HS;
}
void dfs2(int x,int fa=0)
{
    dfn.push_back(x);
    if(hs[x])
    {
        head[hs[x]]=head[x];
        dfs2(hs[x],x);
    }
    for(auto v:edge[x])
    {
        if(v==fa)continue;
        if(v==hs[x])continue;
        dfs2(v,x);
    }
}

int LCA(int a,int b)
{
    while(head[a]!=head[b])
    {
        if(h[head[a]]>h[head[b]])swap(a,b);
        b=F[head[b]];
    }
    return h[a]>h[b]?b:a;
}
```