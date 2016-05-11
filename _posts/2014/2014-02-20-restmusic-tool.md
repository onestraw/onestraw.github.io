---
layout: single
author_profile: true
comments: true
title: RestMusic tool
categories: [Python]
tags: [Python]
---
## 引言
作为一个码农，由于工作性质，需要整天坐在电脑前，长时间坐着，如果不能有规律的锻炼运动，会影响腰椎和颈椎的健康，长此以往，就会严重伤害身体健康。

为做到规律运动，像以前上课一样，有个课间休息，本文设计一个RestMusic工具，它可以每隔一定时间就播放音乐提醒你休息一次，你可以站起来活动活动，减少对腰椎和颈椎的危害。

该工具使用python2.7.6实现，调用pygame模块来播放mp3等格式音乐。  

## RestMusic工具功能

1. 首次使用时，在命令行提示一步步的配置播放列表，将播放列表保存到pickle文件。  
2. 根据命令行参数可以设置播放歌曲的时间间隔，单位分钟，如果没有参数，则默认间隔是45分钟，  
命令如 python RestMusic.py 30 意思是每间隔30分钟提醒一次。  
3. 程序运行时，将播放列表文件的所有音频文件名读入一个列表，然后间隔设置的时间随机播放4分钟。  

## RestMusic工具的不足

1. 没有图形界面  
2. 音频文件不能是中文，没有解决好中文编码问题  


## 播放一首歌曲的步骤

* pygame.mixer.init()
* pygame.mixer.music.load(‘d:\\Song\\Cry On My Shoulder.mp3′)
* pygame.mixer.music.play() #它启动一个线程去播放
* pygame.mixer.music.stop()  #停止一首歌曲

## 源码
{% highlight c %}
'''
功能：间隔固定时间提示休息
使用格式：
	python rest.py 间隔时间(单位:分钟)
'''
import os, sys, string, time, random
import cPickle
import pygame
'''
定义一个课间音乐类，每隔一段时间放一首歌
'''
class RestMusic():
	def __init__(self, intervals=45):
		self.PLAY_LISTS_FILENAME='playlist.pk'
		self.intervals = intervals
		self.playList=[]
	'''
	功能：列出一个目录下的所有文件，不包括目录
	rootDir: 输入目录
	'''
	def ListAllMusic(self, rootDir):
		if os.path.exists(rootDir):
			for files in os.listdir(rootDir):
				filename = os.path.join(rootDir,files)
				if not os.path.isdir(filename):
					print filename
	'''
	功能:手动添加歌曲至播放列表，保存至pk文件
	rootDir:歌曲目录
	'''
	def AddPlayList(self):
		print(u"请输入歌曲目录:")
		rootDir = sys.stdin.readline()
		rootDir = rootDir[0:len(rootDir)-1]
		if os.path.isdir(rootDir):
			self.ListAllMusic(rootDir)
		fp = open(self.PLAY_LISTS_FILENAME, 'wb')
		print(u"输入所配置歌曲目录下的文件名, 输入0结束")
		while 1:
			file = sys.stdin.readline()
			if len(file)==2 and file[0]=='0':
				break;
			filename = os.path.join(rootDir,file)
			cPickle.dump(filename[0:len(filename)-1],fp)
		fp.close()
	'''
	功能：从pk文件中读取播放列表
	'''
	def ReadFromPlayList(self):
		if not os.path.exists(self.PLAY_LISTS_FILENAME):
			print(u"播放列表文件不存在，请配置:")
			self.AddPlayList()
		fp = open(self.PLAY_LISTS_FILENAME, 'rb')
		try:
			while 1:
				self.playList.append(cPickle.load(fp))
		except EOFError:
			print "load finish"
		fp.close()
	'''
	功能：播放一首歌曲
	'''
	def PlayMusic(self, musicFile):
		pygame.mixer.init()
		pygame.mixer.music.load(musicFile)
		pygame.mixer.music.play()
		time.sleep(4*60)  #每首歌曲播放4分钟
		pygame.mixer.music.stop()
	'''
	从播放列表随机播放一首歌曲
	'''
	def RandomPlay(self):
		if len(self.playList) == 0:
			self.ReadFromPlayList()
		try:
			musicNum = len(self.playList)
			rnum = random.randint(0, musicNum-1)
			print(u"正播放%s" %self.playList[rnum])
			self.PlayMusic(self.playList[rnum])
		except IndexError as e:
			print(e)
			sys.exit(1)
	'''
	运行
	'''
	def run(self):
		while 1:
			self.RandomPlay()
			time.sleep(self.intervals*60)
#main
if __name__=='__main__':
	if len(sys.argv)==2:
		interval = string.atoi(sys.argv[1])
		rm = RestMusic(interval)
	else:
		rm = RestMusic()
	rm.run()
{% endhighlight %}
