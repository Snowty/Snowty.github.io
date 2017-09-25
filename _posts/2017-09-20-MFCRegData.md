---
layout: post
title: MFC查询注册表数据
date: 2017-09-20 
tags: Coding MFC Sec
---

### 相关注册表数据

想要通过查询注册表，收集一些系统信息。

1.系统版本信息:	HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion


    ProductName:Windows 10 Enterprise 2015 LTSB

    RegisteredOwner:Snowty

    SystemRoot:C:\WINDOWS

![](/images/posts/2017/09/MFCRegData//1.png) 

2.是否为虚拟机：	HKEY_LOCAL_MACHINE/HARDWARE/DESCRIPTION/System/BIOS/


    SystemProductName:VMware Virtual Platform
   
![](/images/posts/2017/09/MFCRegData//2.png)

3.电脑位数：	HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Environment\

    PROCESSOR_ARCHITECTURE:
		
        x86　　→32位操作系统
		
        AMD64 →64位操作系统
		
        IA64　→64位操作系统（for Itanium-based）

![](/images/posts/2017/09/MFCRegData//3.png)

4.office版本：	HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\

    office2003:11.0
    office2007:12.0
    office2010:14.0
    office2013:15.0
    office2016:16.0
    office2017:17.0


5.国家/时区：	HKEY_CURRENT_USER\Control Panel\International\

    sCountry:中华人民共和国

    LocalName:zh-CN

    sLanguage:CHS

![](/images/posts/2017/09/MFCRegData//4.png)

### MFC查询注册表信息

代码主要参考《精通MFC程序设计》，人民邮电，姚领田，第24章，注册表编程。

整体来说代码很简单，通俗易懂。但是其间出现了宽字节`\0`截断的问题，真心是折腾了好久🙄

使用`RegGetValue`进行查询，详细用法可以查看[MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/ms724868(v=vs.85).aspx)

{% highlight cpp%}
LONG WINAPI RegGetValue(
  _In_        HKEY    hkey,
  _In_opt_    LPCTSTR lpSubKey,
  _In_opt_    LPCTSTR lpValue,
  _In_opt_    DWORD   dwFlags,
  _Out_opt_   LPDWORD pdwType,
  _Out_opt_   PVOID   pvData,
  _Inout_opt_ LPDWORD pcbData
);
{% endhighlight %}

以查询ProductName为例，全部工程代码见[Git](https://github.com/Snowty/MFC_RegData)。

{% highlight cpp %}
void CRegDataDlg::OnClickedQuery()
{
	// TODO: Add your control notification handler code here
	//--------------------------------查询系统版本信息--------------------------
	//查询ProductName
	DWORD cdData=80;	//预设置的数据长度
	LPDWORD lp=&cdData;
	LPBYTE ProductName_Get = new BYTE[80];	//保存查询的数据
	long ret = RegGetValue(HKEY_LOCAL_MACHINE, _T("Software\\Microsoft\\Windows NT\\CurrentVersion"),_T("ProductName"), RRF_RT_REG_SZ, NULL, ProductName_Get, lp);//注册表查询，将结果保存在ProductName_Get中

	//将unicode转换为ASCII
	char ProductName_Get_Buf[100];
	DWORD dBufSize = 80;
	WideCharToMultiByte(CP_UTF8, 0, (LPCWCH)ProductName_Get, -1, ProductName_Get_Buf, dBufSize, NULL, FALSE);//转换后的结果保存在ProductName_Get_Buf中
	m_ProductName = CString(ProductName_Get_Buf);//显示在界面上
    
        UpdateData(false);	//将值赋给控件

	delete[] ProductName_Get;

}
{% endhighlight %}

如果使用`RegQueryValueEx`进行查询，则需要先使用`RegOpenKeyEx`打开注册表：
{% highlight cpp %}
void CRegDataDlg::OnClickedQuery()
{
        HKEY hKEY;	//handler
	LPCTSTR banner_Set = _T("Software\\Microsoft\\Windows NT\\CurrentVersion"); //子健目录
	long retopen = (::RegOpenKeyEx(HKEY_LOCAL_MACHINE,banner_Set,0,KEY_READ,&hKEY));	//打开系统版本注册表
	if(retopen!=ERROR_SUCCESS)
	{
		MessageBox(_T("ERROR:Can not open the hKEY!"));
		return;
	}

        //查询RegisteredOwner
	LPBYTE RegisteredOwner_Get = new BYTE[80];
	DWORD type_2 = REG_SZ;
	DWORD cdData_2 = 80;
	long ret2 = ::RegQueryValueEx(hKEY,_T("RegisteredOwner"),NULL,&type_2,RegisteredOwner_Get,&cdData_2);
	if(ret2!=ERROR_SUCCESS)
	{
		MessageBox(_T("Can not query the Reg"));
		return;
	}
	char dBuf2[100];
	DWORD dBufSize2 = 80;
	WideCharToMultiByte(CP_UTF8, 0, (LPCWCH)RegisteredOwner_Get, -1, dBuf2, dBufSize2, NULL, FALSE);
	m_RegisteredOwner = CString(dBuf2);
        delete[] RegisteredOwner_Get;
	::RegCloseKey(hKEY);
}
{% endhighlight %}
### 宽字节转换

    `WideCharToMultiByte`:此函数是将宽字节转换为ascii，刚好在《加密与解密》这本书的第一章看到过，与此对应的是`MultiByteToWideChar`

    `ASCII`：7位，1 byte。

    `Unicode`：16位，2 byte，所以称为宽字节，将7位的ASCII扩充为16位(0来填充)。

书上的源码没有使用这个函数进行转换，所以导致我的输出只有字符串的第一个字符，例如本来应该输出`Windows 10 Enterprise 2015 LTSB`,但只输出了`W`。

后来经彬彬大神的点拨，让我对`ProductName_Get`的字符串进行逐个输出，然后看到了`\0`的bug。

{% highlight cpp%}
for(int i = 0; i < 20; ++i) 
    putchar(ProductName_Get[i]); 
{% endhighlight %}

可以看到，ProductName_Get[0]='W',ProductName_Get[1]='\0',直接把字符串截断了

![](/images/posts/2017/09/MFCRegData//6.png)
