---
layout: single
author_profile: true
comments: true
title: PyQt实现人机对战五子棋
categories: [Python]
tags: [pyqt, Python]
---
<img src='/assets/images/wuziqi.jpg'>

<strong>一、几个主要函数</strong>

1. score()

对活2，活3，活4等进行打分

2. evalBoard()

在位置（x,y）落子后，就评估这一点对整个棋盘局势的影响，即从这一点开始向四周8个方向，四条线上进行搜索，按形成的连2，连3，活3等进行估分

思路不难，就是实现比较复杂，代码较多

3. isWin()函数：判断棋盘是否分出输赢

该函数和evalBoard()函数类似，都要当前落子位置向四面八方进行搜索

4. computerStep()

预测一步，进行防守和进攻判断

防守估值：假设玩家下在这个位置，能获得多大的优势，取一个优势最大的位置，电脑来抢占这个位置，使玩家失去获得最大优势的机会，记为maxDefend

进攻估值：电脑去试探棋盘中所有空闲的位置，对每个位置进行估分，找出得分最高的位置，记为maxAttack

比较maxAttack和maxDefend，来决定下一步是进攻还是防守

<pre class="lang:python decode:true">    #电脑落子
    def computerStep(self):
        #1.电脑防守,假设玩家下这个位置，能获得很大优势，电脑就要抢占这个位置
        maxD=0
        stepPosD=[0,0]
        for i in range(self.lineNum):
            for j in range(self.lineNum):
                if self.chessBoard[i][j]:
                    continue
                self.chessBoard[i][j]==self.man
                temp=self.evalBoard(i,j,self.man)
                if maxD &lt; temp:
                    maxD=temp
                    stepPosD=[i,j]
                    print "(%d,%d)=%d" %(i,j,temp)
                self.chessBoard[i][j]==0
        #2.电脑进攻，电脑去试探每个步骤，看看优势
        maxA=0
        stepPosA=[0,0]
        for i in range(self.lineNum):
            for j in range(self.lineNum):
                if self.chessBoard[i][j]:
                    continue
                self.chessBoard[i][j]==self.computer
                temp=self.evalBoard(i,j,self.computer)
                if maxA &lt; temp:
                    maxA=temp
                    stepPosA=[i,j]
                    print "(%d,%d)=%d" %(i,j,temp)
                self.chessBoard[i][j]==0
        #3.判断电脑是进攻还是防守
        if maxA &gt; maxD:
            return stepPosA
        else:
            return stepPosD</pre>
            
<h1><strong>二、两个事件</strong></h1>

1. paintEvent

在连连看中已有介绍，不再赘述

<pre class="lang:python decode:true">    def paintEvent(self, event):
        print "this is an event"
        self.painter.begin(self)
        self.drawChessBoard()
        for i in range(self.lineNum):
            for j in range(self.lineNum):
                if self.chessBoard[i][j]!=0:
                    self.drawChessMan(i, j, self.chessBoard[i][j])
        self.painter.end()</pre>

2. mousePressEvent

捕获鼠标左键信号，玩家落子

<pre class="lang:python decode:true">def mousePressEvent(self, event):
        clickPos = event.pos()
        x=self.y(clickPos)
        y=self.x(clickPos)
        #print "coord position(%d,%d)" %(x,y)
        if not self.gameOverFlag and event.button()==QtCore.Qt.LeftButton and not self.chessBoard[x][y]:
            print "left click"
            #玩家下
            self.chessBoard[x][y] = self.man
            self.repaint()
            #判断玩家有没有赢
            if self.isWin(x,y,self.man)==1:
                print "Man Win"
                self.gameOverFlag = 1
                self.showGameOver(self.man)
            else:
                #print "continue"
                #电脑下
                coord = self.computerStep()
                self.chessBoard[coord[0]][coord[1]]=self.computer
                self.repaint()
                #判断电脑有没有赢
                if self.isWin(coord[0],coord[1],self.computer)==1:
                    print "Computer Win"
                    self.gameOverFlag = 1
                    self.showGameOver(self.computer)</pre>

<h1><strong>三、不足</strong></h1>
1. 对AI不熟，没有使用博弈树，AI水平初级；

2. 图形界面略丑，并且界面亮度有下棋时发生变化；

(完整源码：<a href="https://github.com/onestraw/PyWZQ" target="_blank">https://github.com/onestraw/PyWZQ</a>)
