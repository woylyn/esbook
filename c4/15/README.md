# 4.15 ES调用身份证阅读器方案
### 目的
公司使用ES做人事管理系统，手工录入身份证信息太麻烦，还容易出错，于是采购了一台华视CRV-100U身份证阅读器配合ES来使用，效率非常高。
 
>村长补充：神盾ICR-100U也可以用类似方案。

![](./4.15-1.jpg)

### 使用方法和步骤
1. 安装身份证阅读器驱动程序。
2. 将身份证阅读器厂家提供的SDK中的DLL文件复制到EXECL程序的安装目录，我是直接将VB开发的DEMO包里的文件全部复制过去了。
3. 将SDK里的 `termb.lic` 文件复制到C盘根目录。
4. 在ES的模板里使用VBA方式调用身份证阅读器。

![](./4.15-2.jpg)

```vb
Public Declare Function CVR_InitComm Lib "termb.dll" (ByVal Port As Long) As Integer '连接设备
Public Declare Function CVR_CloseComm Lib "termb.dll" () As Integer '关闭设备
Public Declare Function CVR_Authenticate Lib "termb.dll" () As Integer '验证身份证
Public Declare Function CVR_Read_Content Lib "termb.dll" (ByVal Active As Long) As Integer '读取身份证信息
'【机读身份证】按钮代码：
Private Sub CommandButton1_Click()
    Dim wzfile As String
    Dim zpfile As String
    '当前目录绝对路径Application.Path
    wzfile = Application.Path & "\wz.txt"
    zpfile = Application.Path & "\zp.bmp"
    '清除已有的文件
    Dim MyFile As Object
    Set MyFile = CreateObject("Scripting.FileSystemObject")
    If MyFile.FileExists(wzfile) = True Then
        Kill wzfile
    End If
    If MyFile.FileExists(zpfile) = True Then
        Kill zpfile
    End If
    Dim i As Integer
    '扫描USB端口，连接身份证阅读器
    For i = 1001 To 1016
        If CVR_InitComm(i) = 1 Then
            '卡认证
            If CVR_Authenticate() = 0 Then
                 MsgBox "身份证认证失败！"
                 Exit Sub
            End If
            '读取身份证信息
            '1:文字wz.txt,数据xp.wlt,图片zp.bmp;
            '2:文字wz.txt,数据xp.wlt;
            '3:最新住址newadd.txt;
            '4:文字(解码)wz.txt,图片zp.bmp;
            '5:芯片管理号IINSNDN.bin;
            '6:文字，图片文件以设备唯一序列号前五位命名(终端网络环境使用)
            If CVR_Read_Content(4) = 0 Then
                 MsgBox "读取身份证信息失败！"
                 Exit Sub
            End If
            '填充身份证信息到EXECL表格
            '提取身份证文字信息
            If MyFile.FileExists(wzfile) = True Then
                Dim arr(1 To 8)
                Worksheets("员工基本信息").Range("C13").ClearContents
                Worksheets("员工基本信息").Range("G19").ClearContents
                Worksheets("员工基本信息").Range("E13").ClearContents
                Worksheets("员工基本信息").Range("C19:E19").ClearContents
                Worksheets("员工基本信息").Range("E15").ClearContents
                Open wzfile For Input As #1
                i = 1 '初始化计数器
                Do While Not EOF(1)
                    Line Input #1, arr(i)
                    If i = 8 Then
                        Exit Do
                    End If
                i = i + 1
                Loop
                Close #1
                Cells(13, 3).Value = Trim(arr(1)) '姓名
                Cells(19, 7).Value = Trim(arr(3)) '民族
                Cells(13, 5).Value = arr(6) '身份证号
                Cells(19, 3).Value = arr(5) ''地址
                Cells(15, 5).Value = Replace(Right(arr(8), 10), ".", "-") '有效期限
            End If            
            If MyFile.FileExists(zpfile) = True Then            
                '备份身份证照片制作员工工卡
                If MyFile.FileExists(Application.Path & "\" & Trim(arr(1)) & arr(6) & ".bmp") = True Then
                    Kill Application.Path & "\" & Trim(arr(1)) & arr(6) & ".bmp"
                End If
                FileCopy zpfile, "D:\身份证照片\" & Trim(arr(1)) & arr(6) & ".bmp"
                'Name Application.Path & "\zp.bmp" As Application.Path & "\" & arr(1) & ".bmp"
                '提取身份证照片
                zpfile = "D:\身份证照片\" & Trim(arr(1)) & arr(6) & ".bmp"
                Dim oAdd As Object 'ES 对象
                Set oAdd = Application.COMAddIns("ESClient10.Connect").Object
                Cells(2, 7).Select
                oAdd.AddPicture zpfile, 1, 2, 7 ' 插入图片
                Set oAdd = Nothing
            End If
            '关闭身份证读卡器
            CVR_CloseComm
            Exit Sub
        End If
    Next
    MsgBox "未连接身份证阅读器！"
End Sub
```

### 本节贡献者
*@云*

2017-12-5
