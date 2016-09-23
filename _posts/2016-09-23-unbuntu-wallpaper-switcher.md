---
layout: post
title:  "ubuntu桌面图片切换脚本"
date:   2016-09-17 13:16:39
categories: test
---
##shell脚本##

`
#! /bin/bash
usage="
1、将该文件放到放有图片的目录里，图片扩展名要为jpg，能看懂代码的可以自己改图片格式 \n
2、在命令行里cd到此文件所在目录，运行 sh changeWallpaper.sh [xml文件的名字]      说明：xml文件的名字不用写“.xml”，如果提供了xml文件的名字，则跳到第4步\n
3、将格式化的xml保存到一个文本文件里，保存。假设名字为myWallpaper.xml \n
4、命令行执行 sudo gedit /usr/share/gnome-background-properties/precise-wallpapers.xml 非12.04版本的ubuntu是这个目录下的其他名字的xml文件，其中应该包含<wallpaper> \n
5、在precise-wallpapers.xml里的第一个<wallpaper>下仿照第一个增加一个<wallpaper>，不过要去掉<options>那行，<name>随便写，<filename>填myWallpaper.xml的绝对路径 \n
6、在 系统设置->外观 里找到带时钟图标的可变壁纸，其中就有自己刚加的那些图片，单击就好了，也可以再选择图片的缩放方式。 \n
7、需要更换图片请重新执行步骤1~6"

#echo ${usage}
curPath=`pwd`
echo "scaning ${curPath} ..."

#可以调节的参数
#图片格式
picFormat="jpg"
#图片展示时间(单位：秒)
picShowDuration="1795.0"
#更换图片时的过渡时间(单位：秒)
picChangeDuration="5.0"

r="<background>\n\t
		<starttime>\n\t\t
		<year>2009</year>\n\t\t
		<month>08</month>\n\t\t
		<day>04</day>\n\t\t
		<hour>00</hour>\n\t\t
		<minute>00</minute>\n\t\t
		<second>00</second>\n\t
	</starttime>\n\t
	<!-- This animation will start at midnight. -->\n"
staticS="\t<static>\n\t\t<duration>${picShowDuration}</duration>\n\t\t<file>"
staticE="</file>\n\t</static>\n"
transitionS="\t<transition>\n\t\t<duration>${picChangeDuration}</duration>\n\t\t<from>"
transitionM="</from>\n\t\t<to>"
transitionE="</to>\n\t</transition>"
i=0
startPic=""
endPic=""
previousPic=""

for f in `ls $curPath`
do
	pic="${curPath}/${f}"
	if [ -f ${pic} -a "${f##*.}" = ${picFormat} ];then
		echo ${pic}
		i=`expr $i + 1`
		endPic=${pic}
		if [ $i -eq 1 ];then
			startPic=${pic}
		else
			r="${r}${transitionS}${previousPic}${transitionM}${pic}${transitionE}"
		fi
		r="${r}${staticS}${pic}${staticE}"
		previousPic=${pic}
	fi
done
r="${r}${transitionS}${endPic}${transitionM}${startPic}${transitionE}\n</background>"
echo "include $i pictures"
if [ -z $1 ];then
	echo $r
else
	echo $r > "$1.xml"
fi

echo ${usage}
`
