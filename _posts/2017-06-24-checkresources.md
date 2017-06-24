---
layout: post
title: 字符串资源检查
date: 2017-06-24 14:08:36
categories: Python
---
# 背景
安卓项目里有一些目录，它们的名称后缀不同，用以存放不同locale下的资源，比如values-zh-rCN目录下存放简体中文的相关资源，values-zh-rTW目录下存放繁体中文相关资源，而values目录下存放的是缺省的资源，按照国际惯例应该存放英文相关资源。Android用这种方式来实现应用的国际化。

在各种资源当中，字符串资源（一般放在strings.xml里）和字符串数组资源（一般放在arrays.xml里）是比较常见同时也是比较容易出现问题的。

这些问题一般包括

* 默认目录下资源缺失，即其他语言目录下有的资源id在默认目录下没有对应的（英文）版本
* 项目中不同module里存在id一样但内容不一样的资源，据说在打包的时候会合并相同id的资源，这在不同module相同id的资源内容不一致的情况下会有问题，所以要避免不同module里有id相同（类型也相同）的资源
* 同样的id，在不同语言里格式不一致，比如字符串里的占位符数量不同或类型不同
 
上述问题都是比较常见的、致命的、但容易被忽视的。我所在的团队也是在一次次遇到相关问题后才总结出了上述问题的，同时也下决心建立一种机制来避免这些低级问题。

以上的背景催生出了本文要分享的小程序，它主要用来检查strings.xml和arrays.xml中可能存在的问题，它可以检查string、plurals和string-array资源。姑且叫它`字符资源检查器`吧

# 代码清单

以下假设项目有两个module：app和common

```python
#!/usr/bin/python
#coding=utf-8
import xml.dom.minidom
import os
import re
import sys
reload(sys)
sys.setdefaultencoding('utf8')

#2017年05月14日18:10:54
#此文件放在项目根目录下的app目录，如果要放到其他位置，可以修改rootDir的值，rootDir指示项目了根目录的位置
#用于检查资源文件strings.xml和arrays.xml中可能存在的问题
rootDir = '../'

formatPattern = re.compile(r'%(\d*\$)?[a-zA-Z]')

defaultUniqueStringNameDict = {}
defaultUniquePluralsNameDict = {}

def diff(moduleName, type) :
    "查找默认和简体中文之间是否缺失项目"
    fileName = getFileName(type)
    if fileName == '':
        return

    defaultPath = rootDir + moduleName + '/src/main/res/values/' + fileName + '.xml'
    zhCNPath = rootDir + moduleName + '/src/main/res/values-zh-rCN/' + fileName + '.xml'

    if type == 'string' or type == 'plurals':
        defaultSet = getSet(defaultPath, type)
        zhCNSet = getSet(zhCNPath, type)
        # 为第三步准备数据
        if type == 'string':
            defaultUniqueStringNameDict[moduleName] = defaultSet - zhCNSet
        else:
            defaultUniquePluralsNameDict[moduleName] = defaultSet - zhCNSet

        diffSet = zhCNSet - defaultSet
        if len(diffSet) != 0:
            print moduleName + '模块的默认' + fileName + '.xml缺少了一些' + type + ':'
            setErrorColor()
            for name in diffSet:
                print name
            setDefaultColor()
    elif type == 'string-array':
        defaultDict = getDict(defaultPath, type)
        zhCNDict = getDict(zhCNPath, type)
        diffKeySet = set(zhCNDict.keys()) - set(defaultDict.keys())
        if len(diffKeySet) != 0:
             print moduleName + '模块的默认' + fileName + '.xml缺少了一些' + type + ':'
             setErrorColor()
             for stringArrayName in diffKeySet:
                print stringArrayName
        for key in zhCNDict:
            if key not in defaultDict.keys():
                continue
            if zhCNDict[key] != defaultDict[key]:
                print moduleName + '模块的默认' + fileName + '.xml中name为',
                setErrorColor()
                print key,
                setDefaultColor()
                print '的' + type + '中的item数量和简体中文中的不一样！'
        setDefaultColor()
    else:
        return

def getSet(filePath, type) :
    "得到filePath里所有tagName的name集合"
    if os.path.exists(filePath) == False:
        return set()

    dom = xml.dom.minidom.parse(filePath)
    itemList = dom.documentElement.getElementsByTagName(type)
    nameSet = set()
    for item in itemList:
        nameSet.add(item.getAttribute('name'))
    return nameSet

def getDict(filePath, type) :
    '得到字符数组的name和items数量的映射'
    if os.path.exists(filePath) == False:
        return {}

    dict = {}
    dom = xml.dom.minidom.parse(filePath)
    itemList = dom.documentElement.getElementsByTagName(type) # tagName应为string-array
    for item in itemList:
        dict[item.getAttribute('name')] = len(item.getElementsByTagName("item"))
    return dict

def checkUnique(mainModuleName, subModules, types):
    '检测某种资源的唯一性'
    for type in types:
        fileName = getFileName(type)
        filePath1 = rootDir + mainModuleName + '/src/main/res/values/' + fileName + '.xml'
        set0 = getSet(filePath1, type)
            
        for moduleName in subModules:
            filePath2 = rootDir + moduleName + '/src/main/res/values/' + fileName + '.xml'
            set1 = getSet(filePath2, type)
            setIntersection = set0 & set1
            if len(setIntersection) != 0:
                for name in setIntersection:
                    print '构建' + mainModuleName + '模块时' + moduleName + '子模块中存在name为',
                    setWarningColor()
                    print name,
                    setDefaultColor()
                    print '的重复' + type
            set0 = set0 | set1
        
    return

def getName2FormatInfoDict(filePath, module, language, type):
    '得到name和格式化信息的映射'
    if os.path.exists(filePath) == False:
        return {}

    dict = {}
    dom = xml.dom.minidom.parse(filePath)
    itemList = dom.documentElement.getElementsByTagName(type)
    for item in itemList:
        name = item.getAttribute('name')
        if type == 'string':
            value = getContent(item)
            if formatPattern.search(value):
                dict[name] = getFormatInfoDict(value)
        elif type == 'plurals':
            pluralsItems = item.getElementsByTagName('item')
            i = 0
            dict0 = {}
            for pluralsItem in pluralsItems:
                if i == 0:
                    dict0 = getFormatInfoDict(getContent(pluralsItem))
                else:
                    dict1 = getFormatInfoDict(getContent(pluralsItem))
                    # 单个plurals不同item之间的格式化占位符对比
                    compareFormatInfoDict(module, language, type, name, dict0, dict1)
                i = i + 1
            if len(dict0) > 0:
                #这里取第一个item的信息
                dict[name] = dict0
    return dict

def getFormatInfoDict(value):
    '获取格式化占位符的类型和数量的映射信息'
    match = formatPattern.finditer(value)
    dict = {}
    for m in match:
        if m.group() not in dict.keys():
            dict[m.group()] = 1
        else :
            dict[m.group()] = dict[m.group()] + 1
    return dict

def compareFormatInfoDict(module, language, type, name, dict0, dict1):
    '对比单个名字为name的项目的格式化占位符映射信息'
    keys0 = set(dict0.keys())
    keys1 = set(dict1.keys())
    if len(keys0 - keys1) != 0 or len(keys1 - keys0) != 0:
        print '模块' + module + '的' + language + '目录下name为',
        setErrorColor()
        print name,
        setDefaultColor()
        print '的' + type + '在不同语言版本中格式化占位符的种类不匹配'
    for key in keys0 & keys1:
        if dict0[key] != dict1[key]:
            print '模块' + module + '的' + language + '目录下name为',
            setErrorColor()
            print name,
            setDefaultColor()
            print '的' + type + '的格式化占位符' + key + '在不同语言版本中的数量不匹配：' + str(dict0[key]) + ', ' + str(dict1[key])
    return

def checkFormatInfo(modules, languages):
    '检查格式化占位符匹配情况'
    for module in modules:
        i = 0
        stringDict0 = {}
        pluralsDict0 = {}
        for language in languages:
            filePath = rootDir + module + '/src/main/res/values' + ('-' + language if len(language) > 0 else '') + '/strings.xml'
            stringDict = getName2FormatInfoDict(filePath, module, language, 'string')
            pluralsDict = getName2FormatInfoDict(filePath, module, language, 'plurals')
            if i == 0:
                stringDict0 = stringDict
                pluralsDict0 = pluralsDict
            else:
                #经过第一步，默认语言dict里的name是最全的
                for name in stringDict0.keys():
                    if name not in stringDict.keys():
                        # if name not in defaultUniqueStringNameDict[module]:
                        #     print '模块' + module + '的' + language + '目录下缺少name为',
                        #     setWarningColor()
                        #     print name,
                        #     setDefaultColor()
                        #     print '的string'
                        continue
                    compareFormatInfoDict(module, language, 'string', name, stringDict0[name], stringDict[name])
                for name in pluralsDict0.keys():
                    if name not in pluralsDict.keys():
                        # if name not in defaultUniquePluralsNameDict[module]:
                        #     print '模块' + module + '的' + language + '目录下缺少name为',
                        #     setWarningColor()
                        #     print name,
                        #     setDefaultColor()
                        #     print 'plurals'
                        continue
                    compareFormatInfoDict(module, language, 'plurals', name, pluralsDict0[name], pluralsDict[name])
            i = i + 1
    return

def getFileName(type):
    '获得类型所在的文件名'
    if type == 'string' or type == 'plurals':
        return 'strings'
    elif type == 'string-array':
        return 'arrays'
    else:
        return ''

def getContent(item):
    '获取string或plurals里的实际内容'
    str = ''
    for it in item.childNodes:
        if it.nodeName == '#text':
            str += it.nodeValue
        elif it.nodeName == 'xliff:g':
            str += it.firstChild.data
    return str

def setErrorColor():
    print('\033[1;31;40m'),

def setWarningColor():
    print('\033[1;33;40m'),

def setDefaultColor():
    print('\033[0m'),

print '第一步：检测默认文件中是否有缺失'
modules = ['app', 'common']
types = ['string', 'plurals', 'string-array']
for module in modules:
    for type in types:
        diff(module, type)

setDefaultColor()


print '\n\n第二步：检测子module中是否有重复'
mainModules = ['app']
subModules = ['common']
types = ['string', 'plurals', 'string-array']
#setWarningColor();
for module in mainModules:
    checkUnique(module, subModules, types)

setDefaultColor()

print '\n\n第三步：检测格式化占位符匹配情况'#不支持arrays.xml里有格式化占位符
modules = ['app', 'common']
languages = ['', 'bo-rCN', 'zh-rCN', 'zh-rTW']
checkFormatInfo(modules, languages)

print '\n\n扫描结束'
```
