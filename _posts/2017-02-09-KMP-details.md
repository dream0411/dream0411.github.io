---
layout: default
title: KMP算法
---

{{ page.title }}
===

算法问题定义：在字符文本串txt中查找模式串pat，文本串长N，模式串长M，字符有R种取值。

1 普通算法：暴力查找，字符文本中逐个字符匹配，不匹配则回溯；

2 KMP能减少回溯，方法是使用DFA在不匹配时，确定一个能回溯字符数更少的位置；

3 DFA是确定有限自动机，有状态0至M，中间有R种转移条件，即可能的出现的每个字符都是一种转移条件；

4 DFA的使用方式，是定义一个变量X表示当前状态，开始时为0，将文本串中的字符逐个输入DFA，得到新的状态X，由此循环调用，直到当前状态X抵达M(匹配)或文本串已经扫描到结尾(不匹配)；使用DFA搜索的代码如下：

```
public int search(String txt) {

	// simulate operation of DFA on text
	int m = pat.length();
	int n = txt.length();
	int i, j;
	for (i = 0, j = 0; i < n && j < m; i++) {
		j = dfa[txt.charAt(i)][j];
	}
	if (j == m) return i - m;    // found
	return n;                    // not found
}
```

5 由此，剩余问题是怎样构造一个DFA，DFA只与模式串有关，包括模式串的内容，和模式串中的字符的取值范围；比如字符采用ASCII，256个字符，当然文本串中也使用了这个范围的；如果是实际应用中，也可用字节比较，取字节的范围，也是256种；

5.1	因为DFA的状态有0至M种，所以需要知道0至M-1状态下，每个状态遇到每种字符都会转移到何种状态，由此可以定义二维数组，R行，M列；
用法举例：
dfa['a'][0] = 1
表示在状态1下输入字符a，则将状态转移至2；

5.2 构造DFA开始，默认将DFA的所有行列设置为默认设置值0，0表示状态机的起始位置，然后将模式串第一个字符在0状态下的值设置为1，即0状态下遇到模式串的第一个字符时，状态将转换为1；

5.3 开始循环遍历列1～M-1，即状态1到最后一个状态M-1，不处理M，是因为M是最后一个状态，不会再转移至其他状态，这里遍历处理的是会转移到其他状态的状态；

5.4 对于每一列，处理2件事：匹配和不匹配；

5.4.1 匹配的将状态设置为下一个状态，如第1列遇到了正确的字符b，则执行dfa['b'][1] = 2；

5.4.2 不匹配的回退，这一步是关键，我们作如下演绎：

假设文本串是abababaca，模式串为ababac，我们用暴力查找法现进行一些匹配，如：

```
位置:	0 1 2 3 4 5 6 7 8
文本串:	a b a b a b a c a
模式串:	a b a b a c
```

我们现在查找的文本串位置是0，模式串位置是0；此时匹配了开头的ababa，随后的文本串的字符b，与模式串的字符c不匹配了，但是这个不匹配位置的前5个字符都匹配了；
如果按照暴力查找法，我们应该将文本串的位置右移一位，再与模式串从头匹配一次，如：

```
位置:	0 1 2 3 4 5 6 7 8
文本串:	a b a b a b a c a
模式串:	  a b a b a c
```

我们发现，现在需要从位置1开始匹配，由此可总结出这个情况：当模式串的位置j与文本串的位置i不匹配，此时模式串的前缀0～j-1位置和文本串正在匹配的位置之前的j个字符是相同的，即txt[i-j:i] = pat[0:j]，如果按照暴力查找法逐个右移文本串位置，将会经历一个过程，在pat[1:j-1]中查找pat的一个前缀(在pat[1:j-1]中查找是因为txt[i-j:i]等于pat[0:j]，而从1开始是因为右移了一位，j-1是因为刚才在j位置是不匹配的)；这个过程将持续至下面的这个情况：

```
位置:	0 1 2 3 4 5 6 7 8
文本串:	a b a b a b a c a
模式串:	    a b a b a c
```

我们可以看到，到达这个位置时，刚才不匹配的文本串位置5，现在已经可以和模式串的位置3匹配了；由此回顾刚才第一次发现位置5不匹配时，我们希望第二步的逐个字符右移的过程能搜索到的这个前缀尽可能长，如果可能，最好一步就能将文本串的位置5和模式串的位置3直接对齐，而这就是DFA能做到的效果；为什么DFA能达到这个效果和怎样达到这个效果呢？在上面的状态标注出右移逐字符搜索的范围：

```
位置:	0 [1 2 3 4 5] 6 7 8
文本串:	a [b a b a b] a c a
模式串:	  [  a b a b] a c
```

我们可以发现这是个相对于整个文本串和整个模式串的一个子问题，在文本串的一个小片段中查找模式串的一个前缀的小片段，而且这分别就是上面第一次发现不匹配时需要回溯的文本串片段和pat前缀：txt[i-j:i]与pat[0:j]，即是pat[1:j-1]与pat[0:j]，请注意此时我们的DFA已经建立起包含0～j列的状态，由此可归结出，这个子问题就是在已经建立好的模式串的DFA部分，搜索这部分模式串的一个子串(TODO 中缀还是后缀？)，这个子问题与主问题是一样的逻辑，只是文本串和模式串更小；基于这个串的范围其实都是pat，而且j与j-1是递推的关系，所以建立DFA也就是一个迭代的过程，逐列地基于前面的部分建立起来，可以比喻为，建造台阶，每建好一级，就踩上去建更高的一级。

再看上面文本串位置5(也是模式串5)不匹配时回溯后，模式串将在位置3与文本串位置5匹配匹配，也就是模式串达到了状态3，所以对于模式串的DFA状态5，只要从状态3拷贝过来，再将正确字符在该列的状态设置为下一状态j+1即6，该列的构造就完成了，继续处理下一列(因为没有到达末尾，还有下一列需要处理)；这一步可以抽象为，构造列j时，先将可以回溯到的前面的某列X的状态拷贝过来，再将该列正确字符的状态设置为下一状态)。

那么剩下一个问题，怎样知道该将哪一列X拷贝至j列呢？我们先注意一点，DFA的0列是在步骤5.2的初始化时构造好的，遍历是从列1开始，从列2开始pat[1:j-1]才会包含非空的子串；然后从列1构造完成后开始，我们将经历一个DFA构造的迭代过程；为了知道在每个状态匹配到错误的字符后，搜索前面的pat[1:j-1]后能回溯至哪一状态，同时要在该迭代过程中，加入一个搜索的迭代过程(使用X记录这个搜索的状态，与主任务一样，默认状态0)，搜索列1至完成的列的子串pat[1:j-1]；从列1开始，每完成一列后，都将该列的正确字符输入DFA进行查找dfa[pat[j]][X]得到的结果(注意0~j列都已完成，而且X小于j，所以肯定能查到结果)，即是新的状态X(代码：X = dfa[pat[j]][X])，这一状态将用于复制到下一列；KMP的作者们创造了这个精炼的构造过程的算法：

```
// build DFA from pattern
int m = pattern.length;
dfa = new int[R][m]; 
dfa[pattern[0]][0] = 1; 
for (int x = 0, j = 1; j < m; j++) {
    for (int c = 0; c < R; c++) 
        dfa[c][j] = dfa[c][x];     // Copy mismatch cases. 
    dfa[pattern[j]][j] = j+1;      // Set match case. 
    x = dfa[pattern[j]][x];        // Update restart state. 
}
```

下面给出构造DFA的数据轨迹，小括号内是当前X表示的重启列，中扩号是正在设置的列j，大括号是模式串在j列的字符设置的下一个状态：

```
--- init ---
       a    b    a    b    a    c
a:     1    0    0    0    0    0
b:     0    0    0    0    0    0
c:     0    0    0    0    0    0
--- copy and match b ---
       a    b    a    b    a    c
a:    (1)  [1]   0    0    0    0
b:    (0)  {2}   0    0    0    0
c:    (0)  [0]   0    0    0    0
--- copy and match a ---
       a    b    a    b    a    c
a:    (1)   1   {3}   0    0    0
b:    (0)   2   [0]   0    0    0
c:    (0)   0   [0]   0    0    0
--- copy and match b ---
       a    b    a    b    a    c
a:     1   (1)   3   [1]   0    0
b:     0   (2)   0   {4}   0    0
c:     0   (0)   0   [0]   0    0
--- copy and match a ---
       a    b    a    b    a    c
a:     1    1   (3)   1   {5}   0
b:     0    2   (0)   4   [0]   0
c:     0    0   (0)   0   [0]   0
--- copy and match c ---
       a    b    a    b    a    c
a:     1    1    3   (1)   5   [1]
b:     0    2    0   (4)   0   [4]
c:     0    0    0   (0)   0   {6}
```

参考博客：[http://www.voidcn.com/blog/xiangshimoni/article/p-371782.html](http://www.voidcn.com/blog/xiangshimoni/article/p-371782.html)
