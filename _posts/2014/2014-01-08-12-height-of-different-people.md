---
layout: single
author_profile: true
comments: true
title: 12个高矮不同的人
categories: [cprogram]
tags: [C/C++, 笔试]
---
阿里巴巴一道笔试题

<strong>问题描述:  </strong>12个高矮不同的人，排成两排，每排必须是从矮到高排列，而且第二排比对应的第一排的人高，问排列方式有多少种?

这个笔试题，很YD，因为把某个递归关系隐藏得很深。&lt;–这是我看到的提示信息–&gt;

这个提示信息很关键，至少知道往什么方向上思考了，避开很多弯路。

<strong>分析：</strong>

12个人，分成两排，每排都是从矮到高排列，而且第二排的人比站在他前面的人高，如果用一个r[2][6]的数组表示一个排列，那么r[0][0]最矮（小），r[1][5]最高（大）。

<strong>三个数组</strong>

把这12个人的身高存储在数组h[12]中，最终排列结果存储在r[2][6]中，我们每一次排列都从第一列r[][0]开始，然后依次分配下一列。

每次从数组h[]中选两个数放置在数组r[][]中，为了标记数组h[]中已经使用的元素，定义一个辅助数组used[12]用以标记h[]对应位置有没有被使用。

<strong>isPlace()函数</strong>

假定数组r[2][6]的分配顺序为从上到下，从左到右，即首先是r[0][0], r[1][0]，最后是r[0][5], r[1][5]。

每次我们判断在一个位置r[i][j](i=0,1; j=0,1,…,5)能否放置h[k](k=0,1,…,11)中的一个数时，要保证如下条件

used[k] = 0

h[k] &gt; r[i][j-1]  如果i&gt;0

r[i][j] &gt; r[i-1][j]  如果i=1

<strong>嵌套for循环</strong>

数组r[2][6]的每一个位置可以有12个选择（普遍情况，不要去考虑isPlace()失败的发问，r[0][0]只能放最小的，r[1][5]只能放最大的）

我们一次放置一列的两个数，使得从第0列到第col列满足条件。

<strong>C语言+递归+回溯 </strong>

<pre class="lang:c decode:true  ">#include&lt;stdio.h&gt;

int h[12]={1,2,3,4,5,6,7,8,9,10,11,12};
int used[12]={0};
int r[2][6];

int n = sizeof(h)/sizeof(int);
int rows = 2;
static int count;//不同的排列数
//打印输出排列结果
void output()
{
        int k,i;
        for(k=0; k&lt;rows; k++)
        {
                for(i=0; i&lt;n/rows; i++)
                        printf("%d,",r[k][i]);
                printf("\n");
        }
        printf("--------------------\n");
}
//判断h[x],h[y]能否放置在r[0][col], r[1][col]
int isPlace(int x, int y, int col)
{
        if(!used[x] &amp;&amp; !used[y] &amp;&amp; h[x]&lt;h[y] &amp;&amp; (col==0 ||(col&gt;0 &amp;&amp; h[x]&gt;r[0][col-1] &amp;&amp;h[y]&gt;r[1][col-1])))
                return 1;
        return 0;
}
//在1、2排的第col列放置两个适当的数
void arrange(int col)
{
        if(col==n/rows)
        {
                output();
                count ++;
        }
        else
        {
                int i, j;
                for(i=0; i&lt;n; i++)
                {       
                        for(j=0; j&lt;n; j++)
                        {
                                if(!isPlace(i,j,col))
                                        continue;
                                used[i] = 1;
                                used[j] = 1;
                                r[0][col] = h[i];
                                r[1][col] = h[j];

                                arrange(col+1);//递归

                                used[i] = 0;
                                used[j] = 0;//回溯
                        }
                }
                
        }
}

int main()
{
        arrange(0);
        printf("排列方式有 %d 种.\n",count);
        return 0;
}</pre>
结果是132种排列方式。是不是感觉和N-皇后问题的回溯算法很相似，同样有一个isPlace函数，用来决断是否可放置，递归之后有used[i]=0来回溯。

&nbsp;
