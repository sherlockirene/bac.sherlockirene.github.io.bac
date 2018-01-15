---
layout:     post
title:      如何利用或与非运算灵活设置TLS/SSL-for windows
subtitle:   TLS/SSL
date:       2018-01-11
author:     kun
header-img: img/post-bg-tls_ssl.jpg
catalog: true
tags:
    - C++
    - Windows
    - TLS
    - SSL
    - 注册表
---


## 概述
***
在Windows平台下做安全辅助工具时经常会对IE高级选项中的TLS/SSL进行设置，在银行各种安全软件开发中尤为常见。

> _[TLS/SSL 传输层安全性协议 维基百科](https://zh.wikipedia.org/wiki/%E5%82%B3%E8%BC%B8%E5%B1%A4%E5%AE%89%E5%85%A8%E6%80%A7%E5%8D%94%E5%AE%9A)_

## 知识储备
***
此项设置共包含
`TLS 1.0``TLS 1.1``TLS 1.2`
`SSL 2.0``SSL 3.0`

微软将这五项设置全部融合在windows注册表中`HKEY_CURRENT_USER`，`Software\Microsoft\Windows\CurrentVersion\Internet Settings`，`SecureProtocols`项里。
> _[官方参考文档](https://support.microsoft.com/en-us/help/3140245/update-to-enable-tls-1-1-and-tls-1-2-as-a-default-secure-protocols-in)_

例如，假设勾选TLS 1.0，TLS 1.1，TLS 1.2，附图表说明：

|  Item   | Hex | Dec | Oct |     Bin      |
|:-------:|----:|----:|----:|-------------:|
|× SSL 2.0|    8|    8|   10|          1000|
|× SSL 3.0|   20|   32|   40|     0010 0000|
|√ TLS 1.0|   80|  128|  200|     1000 0000|
|√ TLS 1.1|  200|  512| 1000|0010 0000 0000|
|√ TLS 1.2|  800| 2048| 4000|1000 0000 0000|

最后结果为：
* Hex: *0x80 + 0x200 + 0x800 = 0xa80(16)*
* Dec: *128 + 521 + 2048 = 2688(10)*
* Oct: *200 + 1000 + 4000 = 8200(8)*
* Bin: *1000 0000 0000 + 0010 0000 0000 + 1000 0000 = 1010 1000 0000(2)*

## 实例分析
***
* \|：有1为1，全0则0。例：
	3(0011) | 5(0101) = 7(0111)

* &：有0则0，全1位1。例：
	3(0011) & 5(0101) = 1(0001)

* ~：按位取反。
	~3（0011） = -4(1100)

基本知识介绍完毕，我们来尝试运用到实际的安全设置中，假设要求设置:

> 勾选 `√ TLS 1.0`  `TLS 1.1`  `TLS 1.2`
> 
> 弃选 `× SSL 2.0`
> 
> 保留 `- SSL 3.0` (即原来是勾选就保证勾选，原来是弃选，就保证弃选)

我们应该如何去计算最终写入注册表的值呢？

### 分析
> 勾选位：当用 \| 操作，“有1为1”保证最终结果这些位数位必会勾选。

> 弃选位：当先 ~ 操作，使自己的占位是0，再与其他运算做&，“有0则0”保证最终结果这些位数位必会弃选。

> 保留位：应当参与 \| 运算，保证用户原设置会被保留。

即:

`最终设置 = (~弃选位 & 原设置) | 勾选位`

### 完整代码

```
#include <iostream> 
#include <Windows.h> 
#include <tchar.h>
using namespace std;
int main(){
    HKEY hkey;
    LPCTSTR reg_path = _T("Software\\Microsoft\\Windows\\CurrentVersion\\Internet Settings");

    DWORD dwOriginSetting;
    DWORD dwDisableSetting = 0x8;       //× SSL 2.0
    DWORD dwEnableSetting = 0xa80;      //√ TLS 1.0 TLS 1.1 TLS 1.2
    DWORD dwFinalSetting;
    
    //read the origin setting
    if (ERROR_SUCCESS ==  ::RegOpenKeyEx(HKEY_CURRENT_USER, reg_path, 0, KEY_READ, &hkey)){
        
        DWORD dwSize = sizeof(DWORD);
        DWORD dwType = REG_DWORD;

        if(ERROR_SUCCESS != ::RegQueryValueEx(hkey,_T("SecureProtocols"), 0, &dwType, (LPBYTE)&dwOriginSetting,&dwSize)){
            cout << "Error:None value!"<< endl;
        }

        cout << dwOriginSetting << endl;
    }
    ::RegCloseKey(hkey);

    HKEY hkey2;

    dwFinalSetting = (~dwDisableSetting & dwOriginSetting) | dwEnableSetting;
    //set the final setting
    if(ERROR_SUCCESS == ::RegOpenKeyEx(HKEY_CURRENT_USER,reg_path,0,KEY_WRITE,&hkey2)){
        if(ERROR_SUCCESS != ::RegSetValueEx(hkey2, _T("SecureProtocols"), 0, REG_DWORD, (CONST BYTE*)&dwFinalSetting, sizeof(DWORD))){
            cout << "Error:Can't write the value!" << endl;
        }
    }
    ::RegCloseKey(hkey2);

    return 0;

}
```
