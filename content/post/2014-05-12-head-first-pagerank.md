---
title: "Head First PageRank"
date: "2014-05-12"
tags: ["algorithm"]
---
深入浅出PageRank算法 <!--more-->

PageRank算法是谷歌曾经独步天下的“倚天剑”，该算法由Larry Page和Sergey Brin在斯坦福大学读研时发明的，[论文点击下载: The PageRank Citation Ranking: Bringing Order to the Web](http://ilpubs.stanford.edu:8090/422/1/1999-66.pdf)。

本文首先通过一些参考文献引出问题，然后给出了PageRank的几种实现算法，最后将其推广至在MapReduce框架下如何实现PageRank算法。

PageRank的核心思想有2点：   
1.如果一个网页被很多其他网页链接到的话说明这个网页比较重要，也就是pagerank值会相对较高；   
2.如果一个pagerank值很高的网页链接到一个其他的网页，那么被链接到的网页的pagerank值会相应地因此而提高。  

下面是一张来自[WikiPedia](http://en.wikipedia.org/wiki/PageRank)的图，每个球代表一个网页，球的大小反应了网页的pagerank值的大小。指向网页B和网页E的链接很多，所以B和E的pagerank值较高，另外，虽然很少有网页指向C，但是最重要的网页B指向了C，所以C的pagerank值比E还要大。

![image](/images/algos/pagerank-fig.png)

参考内容：   
1.[Wiki about PageRank](http://en.wikipedia.org/wiki/PageRank)   
2.[Google 的秘密- PageRank 彻底解说 中文版](http://www.itlearner.com/good/pagerank_cn.htm)   
3.[数值分析与算法](http://book.douban.com/subject/7161824/) Page 161 应用实例：Google的PageRank算法   
4.[Numeric Methods with Matlab](http://www.mathworks.cn/moler/chapters.html) 或者[中文翻译版本Matlab数值计算](http://book.douban.com/subject/1836464/)   
5.[使用 MapReduce 思想计算 PageRank](http://www.chenjunlu.com/2012/10/pagerank-on-mapreduce/) Page 62 PageRank和马尔可夫链

### 1.问题背景

来自参考内容3

![image](/images/algos/pk-1.png)

### 2.数学建模

来自参考内容3，理解网页连接矩阵$G$，马尔科夫过程("网上冲浪")，转移矩阵$A$，概率$p$为用户点击当前网页中的某个链接地址的概率(一般都为0.85)。

![image](/images/algos/pk-2.png)
![image](/images/algos/pk-3.png)

最后得到一个等式$Ax=x$，这实际上就是求矩阵$A$的特征值为1的特征向量！

下面的内容使用圆盘定理解释了1是矩阵$A$的主特征值，所以我们可以使用幂法来求解。

关于幂法的详细介绍参考另一篇文章[Numerical Methods Using Matlab: 第三章 矩阵特征值和奇异值求解](/blog/2014/04/23/numerical-methods-using-matlab/)

![image](/images/algos/pk-4.png)
![image](/images/algos/pk-5.png)

<!--
![image](/images/algos/pk-6.png)
-->

### 3.求解PageRank

假设有如上图右侧所示的网页链接模型。

(1) 幂法

wiki上有一个PageRank的简便算法，它不考虑转移概率，而是采用的是迭代的方式，每次都更新所有网页的pagerank值，更新的方式就是将每个网页的pagerank值平摊分给它指向的所有网页，每个网页累计所有指向它的网页平摊给它的值作为它该回合的pagerank值，直到全部网页的pagerank值收敛了或者满足一定的阈值条件就停止。

后面的MapReduce框架下PageRank算法的实现就采用了这个思想。考虑转移概率的情况和这个算法类似，乘上一个转移概率再加上一个随机跳转的概率。

![image](/images/algos/pagerank-Simplified-algorithm.png)

根据上面的思想，下面Matlab代码实现可以得到各个网页的PageRank值。

```matlab
n=6;
i=[2 3 4 4 5 6 1 6 1];
j=[1 2 2 3 3 3 4 5 6];
G=sparse(i,j,1,n,n);

% Power method
for j = 1:n
   L{j} = find(G(:,j));
   c(j) = length(L{j});
end

p = .85;
delta = (1-p)/n;
x = ones(n,1)/n;
z = zeros(n,1);
cnt = 0;
while max(abs(x-z)) > .0001
   z = x;
   x = zeros(n,1);
   for j = 1:n
      if c(j) == 0
         x = x + z(j)/n;%转移到任意一个网页
      else
         x(L{j}) = x(L{j}) + z(j)/c(j);%将上次的pagerank值平摊给所有指向的网页
      end
   end
   x = p*x + delta;
   cnt = cnt+1;
end
```
得到的向量$x$保存了各个网页的pagerank值，虽然链接数目一样，但是网页①比网页④和网页⑤都高，而网页②的pagerank值第二高，因为网页①链接到了它上面，相当于沾了网页①的光。

```
x =
    0.2675
    0.2524
    0.1323
    0.1698
    0.0625
    0.1156
```

[这篇文章给出该算法的一个Python版本实现](http://www.chenjunlu.com/2012/09/pagerank-algorithm-implemented-in-python/)，该博主使用第三方模块[python-graph](https://code.google.com/p/python-graph/)，python-graph模块实现了很多图算法，[该模块的使用示例](https://code.google.com/p/python-graph/wiki/Example)，使用前需要先安装，代码如下：

```python
easy_install python-graph-core
easy_install python-graph-dot
```

Python版本的算法实现：

```
# coding=utf-8

# python-graph https://code.google.com/p/python-graph/

# Import graphviz
import graphviz as gv

# Import pygraph
from pygraph.classes.digraph import digraph
from pygraph.readwrite.dot import write

# Define pagerank function
def pagerank(graph, damping_factor=0.85, max_iterations=100, \
             min_delta=0.00001):
    """
    Compute and return the PageRank in an directed graph.

    @type  graph: digraph
    @param graph: Digraph.

    @type  damping_factor: number
    @param damping_factor: PageRank dumping factor.

    @type  max_iterations: number
    @param max_iterations: Maximum number of iterations.

    @type  min_delta: number
    @param min_delta: Smallest variation required for a new iteration.

    @rtype:  Dict
    @return: Dict containing all the nodes PageRank.
    """

    nodes = graph.nodes()
    graph_size = len(nodes)
    if graph_size == 0:
        return {}
    # value for nodes without inbound links
    min_value = (1.0-damping_factor)/graph_size

    # itialize the page rank dict with 1/N for all nodes
    #pagerank = dict.fromkeys(nodes, 1.0/graph_size)
    pagerank = dict.fromkeys(nodes, 1.0)

    for i in range(max_iterations):
        diff = 0 #total difference compared to last iteraction
        # computes each node PageRank based on inbound links
        for node in nodes:
            rank = min_value
            for referring_page in graph.incidents(node):
                rank += damping_factor * pagerank[referring_page] / \
                        len(graph.neighbors(referring_page))

            diff += abs(pagerank[node] - rank)
            pagerank[node] = rank

        print 'This is NO.%s iteration' % (i+1)
        print pagerank
        print ''

        #stop if PageRank has converged
        if diff < min_delta:
            break

    return pagerank

# Graph creation
gr = digraph()

# Add nodes and edges
gr.add_nodes(["1","2","3","4"])

gr.add_edge(("1","2"))
gr.add_edge(("1","3"))
gr.add_edge(("1","4"))
gr.add_edge(("2","3"))
gr.add_edge(("2","4"))
gr.add_edge(("3","4"))
gr.add_edge(("4","2"))

# Draw as PNG
# dot = write(gr)
# gvv = gv.readstring(dot)
# gv.layout(gvv,'dot')
# gv.render(gvv,'png','Model.png')

pagerank(gr)
```

经过32次迭代之后得到的结果如下，和前面的结果一致：

```
This is NO.32 iteration
{'1': 0.2675338708706491, '3': 0.13227261904986046, '2': 0.2524037902400518, '5': 0.062477242064127136, '4': 0.1697488529161491, '6': 0.1155828978186352}
```

(2) 利用马尔可夫矩阵的特殊结构

来自参考内容4，其中$\delta=\frac{1-p}{n}$

![image](/images/algos/pk-8.png)

也就是将矩阵$A$进行分解，并不需要显示求出矩阵$A$，然后便是求解一个线性方程组即可。

```matlab
function x = pagerank1(G)
% PAGERANK1  Google's PageRank modified version 1 - hujiawei

%if nargin < 3, p = .85; end
p=0.85;

% Eliminate any self-referential links

G = G - diag(diag(G));

% c = out-degree, r = in-degree

[n,n] = size(G);
c = sum(G,1);%each row's sum
r = sum(G,2);%each col's sum

% Scale column sums to be 1 (or 0 where there are no out links).

k = find(c~=0);
D = sparse(k,k,1./c(k),n,n);

% Solve (I - p*G*D)*x = e

e = ones(n,1);
I = speye(n,n);
x = (I - p*G*D)\e;

% Normalize so that sum(x) == 1.

x = x/sum(x);
```

(3) 巧妙解法：逆迭代算法

巧妙利用Matlab中的精度误差导致原本是一个奇异矩阵的$I-A$变成一个非奇异矩阵，运行时只是会有些警告提示，但是运行结果和其他算法一样。

![image](/images/algos/pk-9.png)

```
function x = pagerank2(G)
% PAGERANK1  Google's PageRank modified version 2 - hujiawei
% using inverse iteration method

%if nargin < 3, p = .85; end
p=0.85;

% Eliminate any self-referential links

G = G - diag(diag(G));

% c = out-degree, r = in-degree

[n,n] = size(G);
c = sum(G,1);%each row's sum
r = sum(G,2);%each col's sum

% Scale column sums to be 1 (or 0 where there are no out links).

k = find(c~=0);
D = sparse(k,k,1./c(k),n,n);

% Solve (I - p*G*D)*x = e

e = ones(n,1);
I = speye(n,n);
% x = (I - p*G*D)\e;
delta=(1-p)/n;
A=p*G*D+delta;
x=(I-A)\e;

% Normalize so that sum(x) == 1.

x = x/sum(x);
```

最后，附上参考内容4中给出的一份好代码，用于模拟随机冲浪生成矩阵$G$的代码

```
function [U,G] = surfer(root,n)
% SURFER  Create the adjacency graph of a portion of the Web.
%    [U,G] = surfer(root,n) starts at the URL root and follows
%    Web links until it forms an adjacency graph with n nodes.
%    U = a cell array of n strings, the URLs of the nodes.
%    G = an n-by-n sparse matrix with G(i,j)=1 if node j is linked to node i.
%
%    Example:  [U,G] = surfer('http://www.harvard.edu',500);
%    See also PAGERANK.
%
%    This function currently has two defects.  (1) The algorithm for
%    finding links is naive.  We just look for the string 'http:'.
%    (2) An attempt to read from a URL that is accessible, but very slow,
%    might take an unacceptably long time to complete.  In some cases,
%    it may be necessary to have the operating system terminate MATLAB.
%    Key words from such URLs can be added to the skip list in surfer.m.

% Initialize

clf
shg
set(gcf,'doublebuffer','on')
axis([0 n 0 n])
axis square
axis ij
box on
set(gca,'position',[.12 .20 .78 .78])
uicontrol('style','frame','units','normal','position',[.01 .09 .98 .07]);
uicontrol('style','frame','units','normal','position',[.01 .01 .98 .07]);
t1 = uicontrol('style','text','units','normal','position',[.02 .10 .94 .04], ...
   'horiz','left');
t2 = uicontrol('style','text','units','normal','position',[.02 .02 .94 .04], ...
   'horiz','left');
slow = uicontrol('style','toggle','units','normal', ...
   'position',[.01 .24 .07 .05],'string','slow','value',0);
quit = uicontrol('style','toggle','units','normal', ...
   'position',[.01 .17 .07 .05],'string','quit','value',0);

U = cell(n,1);
hash = zeros(n,1);
G = logical(sparse(n,n));
m = 1;
U{m} = root;
hash(m) = hashfun(root);

j = 1;
while j < n & get(quit,'value') == 0

   % Try to open a page.

   try
      set(t1,'string',sprintf('%5d %s',j,U{j}))
      set(t2,'string','');
      drawnow
      page = urlread(U{j});
   catch
      set(t1,'string',sprintf('fail: %5d %s',j,U{j}))
      drawnow
      continue
   end
   if get(slow,'value')
      pause(.25)
   end

   % Follow the links from the open page.

   for f = findstr('http:',page);

      % A link starts with 'http:' and ends with the next quote.

      e = min([findstr('"',page(f:end)) findstr('''',page(f:end))]);
      if isempty(e), continue, end
      url = deblank(page(f:f+e-2));
      url(url<' ') = '!';   % Nonprintable characters
      if url(end) == '/', url(end) = []; end

      % Look for links that should be skipped.

      skips = {'.gif','.jpg','.pdf','.css','lmscadsi','cybernet', ...
               'search.cgi','.ram','www.w3.org', ...
               'scripts','netscape','shockwave','webex','fansonly'};
      skip = any(url=='!') | any(url=='?');
      k = 0;
      while ~skip & (k < length(skips))
         k = k+1;
         skip = ~isempty(findstr(url,skips{k}));
      end
      if skip
         if isempty(findstr(url,'.gif')) & isempty(findstr(url,'.jpg'))
            set(t2,'string',sprintf('skip: %s',url))
            drawnow
            if get(slow,'value')
               pause(.25)
            end
         end
         continue
      end

      % Check if page is already in url list.

      i = 0;
      for k = find(hash(1:m) == hashfun(url))';
         if isequal(U{k},url)
            i = k;
            break
         end
      end

      % Add a new url to the graph there if are fewer than n.

      if (i == 0) & (m < n)
         m = m+1;
         U{m} = url;
         hash(m) = hashfun(url);
         i = m;
      end

      % Add a new link.

      if i > 0
         G(i,j) = 1;
         set(t2,'string',sprintf('%5d %s',i,url))
         line(j,i,'marker','.','markersize',6)
         drawnow
         if get(slow,'value')
            pause(.25)
         end
      end
   end

   j = j+1;
end
delete(t1)
delete(t2)
delete(slow)
set(quit,'string','close','callback','close(gcf)','value',0)

%------------------------

function h = hashfun(url)
% Almost unique numeric hash code for pages already visited.
h = length(url) + 1024*sum(url);
```

### 4.MapReduce框架下PageRank算法的实现

利用前面wiki上的迭代(或者幂法)的思想来实现MapReduce框架下PageRank算法很简单，可以先阅读下参考内容5。

这篇文章[using-mapreduce-to-compute-pagerank](http://michaelnielsen.org/blog/using-mapreduce-to-compute-pagerank/)更加详细，可以参考

以下是我的大数据的一次作业，要求是参考wiki上的简便算法，实现MapReduce框架下的PageRank算法。给的数据集是Twitter的用户之间的关系，可以看做是网页之间的关系，但是助教没要求写代码以及运行这个数据集(有1G多)，所以下面只是一个Python版本的理想可行版本，并没有通过实际大数据集的验证，另外，博主暂时还不太会Python的mapreduce框架中的一些函数，所以实现的是一个简明的可以测试的PageRank算法。

#### 1.输入输出格式

**map函数的输入是<节点，从该节点引出的边列表>，其中节点是一个类，包含了其当前的pagerank值，输出是<节点，反向节点pagerank值/反向节点引出边的总数>；**

**reduce函数的输入是<节点，反向节点pagerank值/反向节点引出边的总数>，输出是<节点，从该节点引出的边列表>，其中节点包含了其更新后的pagerank值。**

伪代码： [一时犯二写了个英文形式的 ]

```
process the data to the form of {node i:[its adjacent node list],...}
while the sum of difference between the last two pagerank values < threshold
	map({node i:[its adjacent node list],...}):
	    map_output={}
	    for every node j in adjacent node list:
	        put or sum up {j:(i, PageRank(i)/length(adjacent node list))} into map_output
	    return map_output

	reduce(map_output):
	    reduce_output={}
	    for every entry {j:(i, PageRank(i)/length(adjacent node list))} in map_output:
	        put or sum up all values pagerank values for node j with its adjacent node list into reduce_output
	    return reduce_output

```

#### 2.示例演示

假设用户1，2，3，4是如下图所示的关系：

![image](/images/others/pagerankdemo.png)

假设有2个mapper(A和B)和1个reducer(C)，初始时4个节点的pagerank值都是0.25

其中，关于用户1和2的数据被mapperA读取并处理，关于用户3和4的数据被mapperB读取并处理 [经验证，即使一个用户的数据是由不同的mapper来读取的，最终收敛到的结果差不多]

map的输入输出结果如下：

![image](/images/others/mapper.png)

reduce的输入输出结果如下，输入是2个mapper的输出，输出的结果中更新了节点的pagerank值

![image](/images/others/reducer.png)

reducer处理完了之后又将它的结果输入给mapper处理，直到迭代的次数超过了设定值或者两次迭代之后得到的所有节点的pagerank值之差的总和(也可以是取二范数)小于设定的阈值。

#### 3.示例的实验结果

(1)首先是使用Matlab采用幂法的方式计算出在p=1.0的情况下示例得到的结果 [它的主要作用是验证后面python版本的正确性]

matlab源码如下：

```
n=4;
i=[2 3 4 3 4 4 1 2];
j=[1 1 1 2 2 3 3 4];
G=sparse(i,j,1,n,n);

[n,n] = size(G);
for j = 1:n
   L{j} = find(G(:,j));
   c(j) = length(L{j});
end

% Power method
p=1.0;
delta = (1-p)/n;
x = ones(n,1)/n;
z = zeros(n,1);
cnt = 0;
while max(abs(x-z)) > .0001
   z = x;
   x = zeros(n,1);
   for j = 1:n
      if c(j) == 0
         x = x + z(j)/n;
      else
         x(L{j}) = x(L{j}) + z(j)/c(j);
      end
   end
   x = p*x + delta;
   cnt = cnt+1;
end
sprintf('pagerank result:')
x
```

结果为：

```
0.1072
0.3571
0.2143
0.3214
```

(2)matlab版本的page rank没有采用mapreduce的思想进行迭代，所以我另外写了一个python版本的利用mapreduce思想实现的pagerank算法(注：我并没有使用python的map和reduce函数去实现，而是使用更加容易明白的实现)，使用的阈值为0.0001，最多迭代的次数为100次。

```
# coding=utf-8

__author__ = 'hujiawei'
__doc__ = 'pagerank mapreduce'

class Node:
    def __init__(self,id,pk):
        self.id=id
        self.pk=pk

def pk_map(map_input):
    map_output={}
    for node,outlinks in map_input.items():
        for link in outlinks:
            size=len(outlinks)
            if link in map_output:
                map_output[link]+=(float)(node.pk)/size
            else:
                map_output[link]=(float)(node.pk)/size
    return map_output

def pk_reduce(reduce_input):
    for result in reduce_input:
        for node,value in result.items():
            node.pk+=value

def pk_clear(nodes):
    for node in nodes:
        node.pk=0

def pk_last(nodes):
    lastnodes=[]
    for node in nodes:
        lastnodes.append(Node(node.id,node.pk))
    return lastnodes

def pk_diff(nodes,lastnodes):
    diff=0
    for i in range(len(nodes)):
        print('node pk %f, last node pk %f ' % (nodes[i].pk, lastnodes[i].pk))
        diff+=abs(nodes[i].pk-lastnodes[i].pk)
    return diff

def pk_test1():
    node1 = Node(1, 0.25)
    node2 = Node(2, 0.25)
    node3 = Node(3, 0.25)
    node4 = Node(4, 0.25)
    nodes = [node1, node2, node3, node4]
    threshold = 0.0001
    max_iters = 100

    for iter_count in range(max_iters):
        iter_count += 1
        lastnodes=pk_last(nodes)
        print('============ map count %d =================' % (iter_count))
        in1 = {node1: [node2, node3, node4], node2: [node3, node4]}
        in2 = {node3: [node1, node4], node4: [node2]}

        mapout1 = pk_map(in1)
        mapout2 = pk_map(in2)

        for node, value in mapout1.items():
            print str(node.id) + ' ' + str(value)

        for node, value in mapout2.items():
            print str(node.id) + ' ' + str(value)

        print('============ reduce count %d =================' % (iter_count))

        reducein = [mapout1, mapout2]
        pk_clear(nodes)
        pk_reduce(reducein)

        for node in nodes:
            print str(node.id) + ' ' + str(node.pk)

        diff=pk_diff(nodes,lastnodes)
        if diff < threshold:
            break

if __name__ == '__main__':
    pk_test1()
```

得到的结果为如下，总共迭代了15次

```
1 0.107138774577
2 0.35712924859
3 0.214296601128
4 0.321435375705
```

上面的结果和Matlab用幂法得到的pagerank值差别很小，可以认为是正确的，所以说明了使用这种mapreduce输入输出格式的正确性。

OK，差不多了，希望对需要理解PageRank算法的人有帮助！ :-)
