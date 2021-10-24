---
layout: post
title: Jitisi print fetch result
---

![image-20210715212457263](\assets\image-20210715212457263.png)

- 一直报错❗

```shell
C:\Users\16593\.virtualenvs\juiti-QRFUgQcO\lib\site-packages\pywinauto\application.py:1076: RuntimeWarning: Application is not loaded correctly (WaitForInputIdle failed)
  warnings.warn('Application is not loaded correctly (WaitForInputIdle failed)', RuntimeWarning)
```

- // 未解决 // - 210720 finished:

add 参数 wait_for_idle=False 即可

```python
chrome_dir = r"D:\\computer\\tools\\googlechrome73\\Google Chrome\\chrome.exe"
app = Application(backend="uia").start(\
    chrome_dir + \
    ' --force-renderer-accessibility'\
    + ' https://map.1tyun.ink:8903/new"', # timeout=20,
    wait_for_idle=False)
```

# ===========================================



```shell
(juiti-QRFUgQcO) D:\mymanchester\CSImage2\VMware\year2_sem1\summer\juiti>python test.py
Traceback (most recent call last):
  File "C:\Users\16593\.virtualenvs\juiti-QRFUgQcO\lib\site-packages\pywinauto\application.py", line 258, in __resolve_control
    criteria)
  File "C:\Users\16593\.virtualenvs\juiti-QRFUgQcO\lib\site-packages\pywinauto\timings.py", line 458, in wait_until_passes
    raise err
pywinauto.timings.TimeoutError

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "test.py", line 20, in <module>
    NewTab.print_control_identifiers() # prints UI elements subtree
  File "C:\Users\16593\.virtualenvs\juiti-QRFUgQcO\lib\site-packages\pywinauto\application.py", line 613, in print_control_identifiers
    this_ctrl = self.__resolve_control(self.criteria)[-1]
  File "C:\Users\16593\.virtualenvs\juiti-QRFUgQcO\lib\site-packages\pywinauto\application.py", line 261, in __resolve_control
    raise e.original_exception
  File "C:\Users\16593\.virtualenvs\juiti-QRFUgQcO\lib\site-packages\pywinauto\timings.py", line 436, in wait_until_passes
    func_val = func(*args, **kwargs)
  File "C:\Users\16593\.virtualenvs\juiti-QRFUgQcO\lib\site-packages\pywinauto\application.py", line 203, in __get_ctrl
    dialog = self.backend.generic_wrapper_class(findwindows.find_element(**criteria[0]))
  File "C:\Users\16593\.virtualenvs\juiti-QRFUgQcO\lib\site-packages\pywinauto\findwindows.py", line 84, in find_element
    elements = find_elements(**kwargs)
  File "C:\Users\16593\.virtualenvs\juiti-QRFUgQcO\lib\site-packages\pywinauto\findwindows.py", line 305, in find_elements
    elements = findbestmatch.find_best_control_matches(best_match, wrapped_elems)
  File "C:\Users\16593\.virtualenvs\juiti-QRFUgQcO\lib\site-packages\pywinauto\findbestmatch.py", line 536, in find_best_control_matches
    raise MatchError(items = name_control_map.keys(), tofind = search_text)
pywinauto.findbestmatch.MatchError: Could not find 'NewTab' in 'dict_keys(['管理员: 命令提示符 - pipenv  shell - python  test.pyDialog', 'Dialog', '管理员: 命令提示符 - pipenv  shell - python  test.py', 'Pane', '新标签页 - Google ChromePane', '新标签页 - Google Chrome', 'Pane0', 'Pane1', 'Pane2', '新标签页 - Google ChromePane0', '新标签页 - Google ChromePane1', '新标签 页 - Google ChromePane2', '新标签页 - Google Chrome0', '新标签页 - Google Chrome1', '新标签页 - Google Chrome2', 'python -  什么是启动谷歌浏览器和通过pywinauto输入网址的最佳方法 - IT工具网 - Google ChromePane', 'Pane3', 'python - 什么是启动谷歌浏览器和通过pywinauto输入网址的最佳方法 - IT工具网 - Google Chrome', 'Pywinauto timings TimeoutError - Google 搜索 - Google ChromePane', 'Pane4', 'Pywinauto timings TimeoutError - Google 搜索 - Google Chrome', '微信Dialog', '微信', 'Dialog0', 'Dialog1', 'Dialog2', '学习卷 (D:)Dialog', 'Dialog3', '学习卷 (D:)', 'juitiDialog', 'juiti', 'Dialog4', 'Program ManagerPane', 'Program Manager', 'Pane5'])'
```



```python
import pywinauto
from pywinauto.application import Application
from pywinauto import Desktop

chrome_dir = r"D:\\computer\\tools\\googlechrome73\\Google Chrome\\chrome.exe"
app = Application().start(\
    chrome_dir + \
    ' --force-renderer-accessibility'\
    + ' https://map.1tyun.ink:8903/new"')

# app.window_(title='New Tab')

# app_new_tab = Application(backend='uia').connect(\
#     path='D:\\computer\\tools\\googlechrome73\\Google Chrome\\chrome.exe',\
#      title_re='New Tab')
# print('app_new_tab', app_new_tab)


print("================1==================")
NewTab = Desktop().window(title=u'1tvideo - Google Chrome')
NewTab.print_control_identifiers() # prints UI elements subtree

print("================2==================")
NewTab.window_text()

print("================3==================")
NewTab.menu_items()

```



输出：



![image-20210715213858046](\assets\image-20210715213858046.png)





终于成功一次？

代码：

```python
import pywinauto
from pywinauto.application import Application
from pywinauto import Desktop

chrome_dir = r"D:\\computer\\tools\\googlechrome73\\Google Chrome\\chrome.exe"
app = Application(backend="uia").start(\
    chrome_dir + \
    ' --force-renderer-accessibility'\
    + ' https://map.1tyun.ink:8903/new"')

print("================1==================")
# NewTab = Desktop(backend="uia").window(title='1tvideo - Google Chrome')
NewTab = Desktop(backend="uia").window(title_re='1tvideo')
NewTab.print_control_identifiers() # prints UI elements subtree
print("================2==================")
# NewTab.window_text()
print("================3==================")
# NewTab.menu_items()
NewTab.maximize()   ### 此项成功！看到他执行了

print("before click")
''' mouse event '''
# pywinauto.mouse.click(button='left', coords=(450, 600))
pywinauto.mouse.click(button='left', coords=(450, 600))
print("after click")

# # print('NewTab', NewTab)
```



输出：

```shell
(juiti-QRFUgQcO) D:\mymanchester\CSImage2\VMware\year2_sem1\summer\juiti>python test.py
C:\Users\16593\.virtualenvs\juiti-QRFUgQcO\lib\site-packages\pywinauto\application.py:1076: RuntimeWarning: Application is not loaded correctly (WaitForInputIdle failed)
  warnings.warn('Application is not loaded correctly (WaitForInputIdle failed)', RuntimeWarning)
================1==================
Control Identifiers:

Pane - '1tvideo - 摄像头或麦克风正在录制/录音 - Google Chrome'    (L-57, T12, R1530, B1029)
['1tvideo - 摄像头或麦克风正在录制/录音 - Google ChromePane', 'Pane', '1tvideo - 摄像头或麦克风正在录制/录音 - Google Chrome', 'Pane0', 'Pane1']
child_window(title="1tvideo - 摄像头或麦克风正在录制/录音 - Google Chrome", control_type="Pane")
   |
   | Document - '          视频 视频      高清       视频 视频          本人                                                         分享 1-1 map.1tyun.ink:8903/new  密码：  未发现设备  复制 •添加密码   邀请您参加会议.\n参加会议:\nhttps://map.1tyun.ink:8903/new\n            '    (L-46, T132, R1520, B1019)
   | ['Document', '          视频 视频      高清   \ue920    视频 视频    \ue916      本人                       \ue917   \ue91e   \ue906     \ue910   \ue905   \ue918   \ue92e   \ue145   \ue922    \ue922   分享 1-1\xa0map.1tyun.ink:8903/new  密码：\xa0 未发现设备  复制 •添加密码   邀请您参加会议.\n参加会议:\nhttps://map.1tyun.ink:8903/new\n   \ue5d4         ', '          视频 视频      高清   \ue920    视频 视频    \ue916      本人                       \ue917   \ue91e   \ue906     \ue910   \ue905   \ue918   \ue92e   \ue145   \ue922    \ue922   分享 1-1\xa0map.1tyun.ink:8903/new  密码：\xa0 未发现设备  复制 •添加密码    邀请您参加会议.\n参加会议:\nhttps://map.1tyun.ink:8903/new\n   \ue5d4         Document']
   | child_window(title="          视频 视频      高清       视频 视频          本人                                                        分享 1-1 map.1tyun.ink:8903/new  密码：  未发现设备  复制 •添加密码   邀请您参加会议.\n参加会议:\nhttps://map.1tyun.ink:8903/new\n            ", auto_id="-1750543232", control_type="Document")
   |    |
   |    | Toolbar - '视频'    (L-52, T132, R1524, B1019)
   |    | ['视频', '视频Toolbar', 'Toolbar', '视频0', '视频1', '视频Toolbar0', '视频Toolbar1', 'Toolbar0', 'Toolbar1']
   |    | child_window(title="视频", control_type="ToolBar")
   |    |
   |    | Toolbar - '视频'    (L-52, T132, R1524, B1019)
   |    | ['视频2', '视频Toolbar2', 'Toolbar2']
   |    | child_window(title="视频", control_type="ToolBar")
   |    |
   |    | Static - '高清'    (L1244, T196, R1284, B218)
   |    | ['高清Static', '高清', 'Static', 'Static0', 'Static1']
   |    | child_window(title="高清", control_type="Text")
   |    |
   |    | Button - ''    (L1483, T879, R1515, B906)
   |    | ['\ue920', 'Button', '\ue920Button', 'Button0', 'Button1']
   |    | child_window(title="", auto_id="toggleFilmstripButton", control_type="Button")
   |    |
   |    | Toolbar - '视频'    (L1328, T160, R1509, B262)
   |    | ['视频3', '视频Toolbar3', 'Toolbar3']
   |    | child_window(title="视频", control_type="ToolBar")
   |    |
   |    | Toolbar - '视频'    (L1328, T160, R1509, B262)
   |    | ['视频4', '视频Toolbar4', 'Toolbar4']
   |    | child_window(title="视频", control_type="ToolBar")
   |    |
   |    | Static - ''    (L1481, T234, R1501, B255)
   |    | ['\ue916Static', '\ue916', 'Static2']
   |    | child_window(title="", control_type="Text")
   |    |
   |    | Static - '本人'    (L1405, T203, R1441, B224)
   |    | ['本人Static', 'Static3', '本人']
   |    | child_window(title="本人", control_type="Text")
   |    |
   |    | Static - ''    (L1, T941, R37, B977)
   |    | ['\ue917', '\ue917Static', 'Static4']
   |    | child_window(title="", control_type="Text")
   |    |
   |    | Static - ''    (L85, T941, R121, B977)
   |    | ['\ue91eStatic', '\ue91e', 'Static5']
   |    | child_window(title="", control_type="Text")
   |    |
   |    | Static - ''    (L169, T941, R205, B977)
   |    | ['\ue906', '\ue906Static', 'Static6']
   |    | child_window(title="", control_type="Text")
   |    |
   |    | Static - ''    (L622, T942, R658, B979)
   |    | ['\ue910Static', '\ue910', 'Static7']
   |    | child_window(title="", control_type="Text")
   |    |
   |    | Static - ''    (L718, T941, R754, B977)
   |    | ['\ue905Static', '\ue905', 'Static8']
   |    | child_window(title="", control_type="Text")
   |    |
   |    | Static - ''    (L814, T942, R850, B979)
   |    | ['\ue918', '\ue918Static', 'Static9']
   |    | child_window(title="", control_type="Text")
   |    |
   |    | Static - ''    (L1183, T941, R1219, B977)
   |    | ['\ue92e', '\ue92eStatic', 'Static10']
   |    | child_window(title="", control_type="Text")
   |    |
   |    | Static - ''    (L1267, T941, R1303, B977)
   |    | ['\ue145', 'Static11', '\ue145Static']
   |    | child_window(title="", control_type="Text")
   |    |
   |    | Static - ''    (L1351, T941, R1387, B977)
   |    | ['\ue922', '\ue922Static', 'Static12', '\ue9220', '\ue9221', '\ue922Static0', '\ue922Static1']
   |    | child_window(title="", control_type="Text")
   |    |
   |    | Custom - ' 分享 1-1 map.1tyun.ink:8903/new 密码：   未发现设备 复制•添加密码'    (L1000, T717, R1411, B917)
   |    | ['\ue922 分享 1-1\xa0map.1tyun.ink:8903/new 密码： \xa0 未发现设备 复制•添加密码Custom', 'Custom', '\ue922 分享 1-1\xa0map.1tyun.ink:8903/new 密码 ： \xa0 未发现设备 复制•添加密码']
   |    | child_window(title=" 分享 1-1 map.1tyun.ink:8903/new 密码：   未发现设备 复制•添加密码", control_type="Custom")
   |    |    |
   |    |    | Static - ''    (L1036, T744, R1060, B769)
   |    |    | ['\ue9222', '\ue922Static2', 'Static13']
   |    |    | child_window(title="", control_type="Text")
   |    |    |
   |    |    | Static - '分享'    (L1096, T744, R1138, B768)
   |    |    | ['分享', '分享Static', 'Static14']
   |    |    | child_window(title="分享", control_type="Text")
   |    |    |
   |    |    | Static - '1-1'    (L1096, T789, R1127, B813)
   |    |    | ['Static15', '1-1Static', '1-1']
   |    |    | child_window(title="1-1", control_type="Text")
   |    |    |
   |    |    | Static - ' '    (L1126, T789, R1133, B813)
   |    |    | ['\xa0', '\xa0Static', 'Static16', '\xa00', '\xa01', '\xa0Static0', '\xa0Static1']
   |    |    | child_window(title=" ", control_type="Text")
   |    |    |
   |    |    | Hyperlink - 'map.1tyun.ink:8903/new'    (L1132, T789, R1360, B813)
   |    |    | ['map.1tyun.ink:8903/new', 'Hyperlink', 'map.1tyun.ink:8903/newHyperlink', 'Hyperlink0', 'Hyperlink1']
   |    |    | child_window(title="map.1tyun.ink:8903/new", control_type="Hyperlink")
   |    |    |
   |    |    | Static - '密码：'    (L1096, T819, R1159, B843)
   |    |    | ['密码：Static', '密码：', 'Static17']
   |    |    | child_window(title="密码：", control_type="Text")
   |    |    |
   |    |    | Static - ' '    (L1159, T819, R1165, B843)
   |    |    | ['\xa02', '\xa0Static2', 'Static18']
   |    |    | child_window(title=" ", control_type="Text")
   |    |    |
   |    |    | Static - '未发现设备'    (L1164, T819, R1270, B843)
   |    |    | ['未发现设备', '未发现设备Static', 'Static19']
   |    |    | child_window(title="未发现设备", control_type="Text")
   |    |    |
   |    |    | Hyperlink - '复制'    (L1096, T865, R1138, B889)
   |    |    | ['复制', 'Hyperlink2', '复制Hyperlink']
   |    |    | child_window(title="复制", control_type="Hyperlink")
   |    |    |
   |    |    | Static - '•'    (L1153, T858, R1165, B895)
   |    |    | ['•', 'Static20', '•Static']
   |    |    | child_window(title="•", control_type="Text")
   |    |    |
   |    |    | Hyperlink - '添加密码'    (L1179, T865, R1264, B889)
   |    |    | ['Hyperlink3', '添加密码', '添加密码Hyperlink']
   |    |    | child_window(title="添加密码", control_type="Hyperlink")
   |    |    |
   |    |    | Edit - ''    (L1036, T741, R1279, B810)
   |    |    | ['Edit', '\ue922Edit', 'Edit0', 'Edit1']
   |    |
   |    | Static - ''    (L1435, T941, R1471, B977)
   |    | ['\ue5d4Static', '\ue5d4', 'Static21']
   |    | child_window(title="", control_type="Text")
   |
   | Pane - ''    (L-46, T12, R1519, B1018)
   | ['Pane2']
   |
   | TitleBar - ''    (L0, T0, R0, B0)
   | ['TitleBar']
   |    |
   |    | Menu - '系统'    (L-46, T23, R-13, B56)
   |    | ['Menu', '系统', '系统Menu', '系统0', '系统1']
   |    | child_window(title="系统", auto_id="MenuBar", control_type="MenuBar")
   |    |    |
   |    |    | MenuItem - '系统'    (L-46, T23, R-13, B56)
   |    |    | ['系统MenuItem', '系统2', 'MenuItem', 'MenuItem0', 'MenuItem1']
   |    |    | child_window(title="系统", control_type="MenuItem")
   |    |
   |    | Button - '最小化'    (L0, T0, R0, B0)
   |    | ['Button2', '最小化', '最小化Button', '最小化0', '最小化1', '最小化Button0', '最小化Button1']
   |    | child_window(title="最小化", control_type="Button")
   |    |
   |    | Button - '最大化'    (L0, T0, R0, B0)
   |    | ['最大化', '最大化Button', 'Button3', '最大化0', '最大化1', '最大化Button0', '最大化Button1']
   |    | child_window(title="最大化", control_type="Button")
   |    |
   |    | Button - '关闭'    (L0, T0, R0, B0)
   |    | ['关闭Button', 'Button4', '关闭', '关闭Button0', '关闭Button1', '关闭0', '关闭1']
   |    | child_window(title="关闭", control_type="Button")
   |
   | Pane - 'Google Chrome'    (L-47, T12, R1520, B1019)
   | ['Google Chrome', 'Pane3', 'Google ChromePane']
   | child_window(title="Google Chrome", control_type="Pane")
   |    |
   |    | Pane - ''    (L-47, T12, R1520, B1019)
   |    | ['Pane4']
   |    |    |
   |    |    | Button - '最小化'    (L1314, T13, R1382, B57)
   |    |    | ['Button5', '最小化2', '最小化Button2']
   |    |    | child_window(title="最小化", control_type="Button")
   |    |    |
   |    |    | Button - '最大化'    (L1381, T13, R1451, B57)
   |    |    | ['最大化2', '最大化Button2', 'Button6']
   |    |    | child_window(title="最大化", control_type="Button")
   |    |    |
   |    |    | Button - '关闭'    (L1450, T13, R1520, B57)
   |    |    | ['关闭Button2', 'Button7', '关闭2']
   |    |    | child_window(title="关闭", control_type="Button")
   |    |
   |    | Pane - ''    (L-47, T25, R1520, B1019)
   |    | ['Pane5']
   |    |    |
   |    |    | Pane - ''    (L-47, T25, R1520, B132)
   |    |    | ['Pane6']
   |    |    |    |
   |    |    |    | TabControl - ''    (L-47, T25, R1314, B78)
   |    |    |    | ['TabControl', 'TabControl新标签页']
   |    |    |    |    |
   |    |    |    |    | TabItem - '1tvideo - 摄像头或麦克风正在录制/录音'    (L-47, T25, R338, B78)
   |    |    |    |    | ['1tvideo - 摄像头或麦克风正在录制/录音', '1tvideo - 摄像头或麦克风正在录制/录音TabItem', 'TabItem']
   |    |    |    |    | child_window(title="1tvideo - 摄像头或麦克风正在录制/录音", control_type="TabItem")
   |    |    |    |    |    |
   |    |    |    |    |    | Button - '关闭'    (L283, T25, R338, B78)
   |    |    |    |    |    | ['关闭Button3', 'Button8', '关闭3']
   |    |    |    |    |    | child_window(title="关闭", control_type="Button")
   |    |    |    |    |
   |    |    |    |    | Button - '新标签页'    (L325, T25, R392, B72)
   |    |    |    |    | ['新标签页Button', 'Button9', '新标签页']
   |    |    |    |    | child_window(title="新标签页", control_type="Button")
   |    |    |    |
   |    |    |    | Pane - ''    (L-47, T76, R1520, B132)
   |    |    |    | ['Pane7']
   |    |    |    |    |
   |    |    |    |    | Button - '后退'    (L-35, T82, R8, B125)
   |    |    |    |    | ['后退', '后退Button', 'Button10']
   |    |    |    |    | child_window(title="后退", control_type="Button")
   |    |    |    |    |
   |    |    |    |    | Button - '前进'    (L13, T82, R56, B125)
   |    |    |    |    | ['前进Button', '前进', 'Button11']
   |    |    |    |    | child_window(title="前进", control_type="Button")
   |    |    |    |    |
   |    |    |    |    | Button - '重新加载'    (L61, T82, R104, B125)
   |    |    |    |    | ['重新加载', '重新加载Button', 'Button12']
   |    |    |    |    | child_window(title="重新加载", control_type="Button")
   |    |    |    |    |
   |    |    |    |    | MenuItem - '查看网站信息'    (L118, T85, R167, B122)
   |    |    |    |    | ['查看网站信息', 'MenuItem2', '查看网站信息MenuItem']
   |    |    |    |    | child_window(title="查看网站信息", control_type="MenuItem")
   |    |    |    |    |
   |    |    |    |    | Edit - '地址和搜索栏'    (L162, T85, R1304, B122)
   |    |    |    |    | ['Edit2']
   |    |    |    |    | child_window(title="地址和搜索栏", control_type="Edit")
   |    |    |    |    |
   |    |    |    |    | Button - '此网页正在使用您的摄像头和麦克风。'    (L1306, T85, R1355, B122)
   |    |    |    |    | ['Button13', '此网页正在使用您的摄像头和麦克风。Button', '此网页正在使用您的摄像头和麦克风。']
   |    |    |    |    | child_window(title="此网页正在使用您的摄像头和麦克风。", control_type="Button")
   |    |    |    |    |
   |    |    |    |    | Pane - ''    (L0, T0, R0, B0)
   |    |    |    |    | ['Pane8']
   |    |    |    |    |
   |    |    |    |    | Button - '为此页添加书签'    (L1354, T85, R1403, B122)
   |    |    |    |    | ['为此页添加书签', 'Button14', '为此页添加书签Button']
   |    |    |    |    | child_window(title="为此页添加书签", control_type="Button")
   |    |    |    |    |
   |    |    |    |    | Pane - ''    (L114, T81, R1407, B126)
   |    |    |    |    | ['Pane9']
   |    |    |    |    |
   |    |    |    |    | Button - '当前用户'    (L1417, T82, R1460, B125)
   |    |    |    |    | ['当前用户Button', 'Button15', '当前用户']
   |    |    |    |    | child_window(title="当前用户", control_type="Button")
   |    |    |    |    |
   |    |    |    |    | MenuItem - 'Chrome'    (L1465, T82, R1508, B125)
   |    |    |    |    | ['Chrome', 'MenuItem3', 'ChromeMenuItem']
   |    |    |    |    | child_window(title="Chrome", control_type="MenuItem")
   |    |    |
   |    |    | Pane - ''    (L-47, T132, R1520, B1019)
   |    |    | ['Pane10', '\ue917Pane']
   |    |    |
   |    |    | Pane - ''    (L0, T0, R0, B0)
   |    |    | ['Pane11']
================2==================
================3==================
before click
after click
```

