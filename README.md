# 🔓 Excel Sheet Password Remover (VBA)

> Remove worksheet protection passwords from any Excel file — no third-party tools needed.

---

## 📌 Overview

A lightweight Excel VBA macro that **removes sheet protection passwords** from any `.xlsx` or `.xls` workbook — even when you've forgotten or lost the password.

The macro works by exploiting a known vulnerability in Excel's sheet protection algorithm, generating a valid "backdoor" password that Excel accepts as a substitute for the original one.

> ⚠️ **Disclaimer:** This tool is intended for **legitimate use only** — such as recovering access to your own files. Do not use it on files you don't own or have permission to access.

---

## ✨ Features

- ✅ Works on any Excel worksheet protected with a password
- ✅ No installation required — pure VBA, runs inside Excel
- ✅ Supports both `.xls` and `.xlsx` formats
- ✅ Processes all sheets in a workbook at once
- ✅ No internet connection needed — 100% offline

---


## 💻 The Code

```vba
Option Explicit

Sub RemoveProtection()

Dim dialogBox As FileDialog
Dim sourceFullName As String
Dim sourceFilePath As String
Dim sourceFileName As String
Dim sourceFileType As String
Dim newFileName As Variant
Dim tempFileName As String
Dim zipFilePath As Variant
Dim oApp As Object
Dim FSO As Object
Dim xmlSheetFile As String
Dim xmlFile As Integer
Dim xmlFileContent As String
Dim xmlStartProtectionCode As Double
Dim xmlEndProtectionCode As Double
Dim xmlProtectionString As String

'Open dialog box to select a file
Set dialogBox = Application.FileDialog(msoFileDialogFilePicker)
dialogBox.AllowMultiSelect = False
dialogBox.Title = "Select file to remove protection from"

If dialogBox.Show = -1 Then
    sourceFullName = dialogBox.SelectedItems(1)
Else
    Exit Sub
End If

'Get folder path, file type and file name from the sourceFullName
sourceFilePath = Left(sourceFullName, InStrRev(sourceFullName, "\"))
sourceFileType = Mid(sourceFullName, InStrRev(sourceFullName, ".") + 1)
sourceFileName = Mid(sourceFullName, Len(sourceFilePath) + 1)
sourceFileName = Left(sourceFileName, InStrRev(sourceFileName, ".") - 1)

'Use the date and time to create a unique file name
tempFileName = "Temp" & Format(Now, " dd-mmm-yy h-mm-ss")

'Copy and rename original file to a zip file with a unique name
newFileName = sourceFilePath & tempFileName & ".zip"
On Error Resume Next
FileCopy sourceFullName, newFileName

If Err.Number <> 0 Then
    MsgBox "Unable to copy " & sourceFullName & vbNewLine _
        & "Check the file is closed and try again"
    Exit Sub
End If
On Error GoTo 0

'Create folder to unzip to
zipFilePath = sourceFilePath & tempFileName & "\"
MkDir zipFilePath

'Extract the files into the newly created folder
Set oApp = CreateObject("Shell.Application")
oApp.Namespace(zipFilePath).CopyHere oApp.Namespace(newFileName).items

'loop through each file in the \xl\worksheets folder of the unzipped file
xmlSheetFile = Dir(zipFilePath & "\xl\worksheets\*.xml*")
Do While xmlSheetFile <> ""

    'Read text of the file to a variable
    xmlFile = FreeFile
    Open zipFilePath & "xl\worksheets\" & xmlSheetFile For Input As xmlFile
    xmlFileContent = Input(LOF(xmlFile), xmlFile)
    Close xmlFile

    'Manipulate the text in the file
    xmlStartProtectionCode = 0
    xmlStartProtectionCode = InStr(1, xmlFileContent, "<sheetProtection")

    If xmlStartProtectionCode > 0 Then

        xmlEndProtectionCode = InStr(xmlStartProtectionCode, _
            xmlFileContent, "/>") + 2 '"/>" is 2 characters long
        xmlProtectionString = Mid(xmlFileContent, xmlStartProtectionCode, _
            xmlEndProtectionCode - xmlStartProtectionCode)
        xmlFileContent = Replace(xmlFileContent, xmlProtectionString, "")

    End If

    'Remove Range Protection
    xmlStartProtectionCode = 0
    xmlStartProtectionCode = InStr(1, xmlFileContent, "<protectedRanges")

    If xmlStartProtectionCode > 0 Then

        xmlEndProtectionCode = InStr(xmlStartProtectionCode, _
            xmlFileContent, "</protectedRanges>") + 18 '"</protectedRanges>" is 18 characters long
        xmlProtectionString = Mid(xmlFileContent, xmlStartProtectionCode, _
            xmlEndProtectionCode - xmlStartProtectionCode)
        xmlFileContent = Replace(xmlFileContent, xmlProtectionString, "")

    End If

    'Output the text of the variable to the file
    xmlFile = FreeFile
    Open zipFilePath & "xl\worksheets\" & xmlSheetFile For Output As xmlFile
    Print #xmlFile, xmlFileContent
    Close xmlFile

    'Loop to next xmlFile in directory
    xmlSheetFile = Dir

Loop

'Read text of the xl\workbook.xml file to a variable
xmlFile = FreeFile
Open zipFilePath & "xl\workbook.xml" For Input As xmlFile
xmlFileContent = Input(LOF(xmlFile), xmlFile)
Close xmlFile

'Manipulate the text in the file to remove the workbook protection
xmlStartProtectionCode = 0
xmlStartProtectionCode = InStr(1, xmlFileContent, "<workbookProtection")
If xmlStartProtectionCode > 0 Then

    xmlEndProtectionCode = InStr(xmlStartProtectionCode, _
        xmlFileContent, "/>") + 2 ''"/>" is 2 characters long
    xmlProtectionString = Mid(xmlFileContent, xmlStartProtectionCode, _
        xmlEndProtectionCode - xmlStartProtectionCode)
    xmlFileContent = Replace(xmlFileContent, xmlProtectionString, "")

End If

'Manipulate the text in the file to remove the modify password
xmlStartProtectionCode = 0
xmlStartProtectionCode = InStr(1, xmlFileContent, "<fileSharing")
If xmlStartProtectionCode > 0 Then

    xmlEndProtectionCode = InStr(xmlStartProtectionCode, xmlFileContent, _
        "/>") + 2 '"/>" is 2 characters long
    xmlProtectionString = Mid(xmlFileContent, xmlStartProtectionCode, _
        xmlEndProtectionCode - xmlStartProtectionCode)
    xmlFileContent = Replace(xmlFileContent, xmlProtectionString, "")

End If

'Output the text of the variable to the file
xmlFile = FreeFile
Open zipFilePath & "xl\workbook.xml" & xmlSheetFile For Output As xmlFile
Print #xmlFile, xmlFileContent
Close xmlFile

'Create empty Zip File
Open sourceFilePath & tempFileName & ".zip" For Output As #1
Print #1, Chr$(80) & Chr$(75) & Chr$(5) & Chr$(6) & String(18, 0)
Close #1

'Move files into the zip file
oApp.Namespace(sourceFilePath & tempFileName & ".zip").CopyHere _
oApp.Namespace(zipFilePath).items
'Keep script waiting until Compressing is done
On Error Resume Next
Do Until oApp.Namespace(sourceFilePath & tempFileName & ".zip").items.Count = _
    oApp.Namespace(zipFilePath).items.Count
    Application.Wait (Now + TimeValue("0:00:01"))
Loop
On Error GoTo 0

'Delete the files & folders created during the sub
Set FSO = CreateObject("scripting.filesystemobject")
FSO.deletefolder sourceFilePath & tempFileName

'Rename the final file back to an xlsx file
Name sourceFilePath & tempFileName & ".zip" As sourceFilePath & sourceFileName _
& "_" & Format(Now, "dd-mmm-yy h-mm-ss") & "." & sourceFileType

'Show message box
MsgBox "The workbook and worksheet protection passwords have been removed.", _
vbInformation + vbOKOnly, Title:="Password protection"

End Sub
```
---

# VBA Project protected

```vba
Option Explicit

Private Const PAGE_EXECUTE_READWRITE = &H40

Private Declare PtrSafe Sub MoveMemory Lib "kernel32" Alias "RtlMoveMemory" _
    (Destination As LongPtr, Source As LongPtr, ByVal Length As LongPtr)

Private Declare PtrSafe Function VirtualProtect Lib "kernel32" (lpAddress As LongPtr, _
    ByVal dwSize As LongPtr, ByVal flNewProtect As LongPtr, lpflOldProtect As LongPtr) As LongPtr

Private Declare PtrSafe Function GetModuleHandleA Lib "kernel32" (ByVal lpModuleName As String) As LongPtr

Private Declare PtrSafe Function GetProcAddress Lib "kernel32" (ByVal hModule As LongPtr, _
    ByVal lpProcName As String) As LongPtr

Private Declare PtrSafe Function DialogBoxParam Lib "user32" Alias "DialogBoxParamA" (ByVal hInstance As LongPtr, _
    ByVal pTemplateName As LongPtr, ByVal hWndParent As LongPtr, _
    ByVal lpDialogFunc As LongPtr, ByVal dwInitParam As LongPtr) As Integer

Dim HookBytes(0 To 11) As Byte
Dim OriginBytes(0 To 11) As Byte
Dim pFunc As LongPtr
Dim Flag As Boolean

Private Function GetPtr(ByVal Value As LongPtr) As LongPtr
GetPtr = Value
End Function

Public Sub RecoverBytes()
If Flag Then MoveMemory ByVal pFunc, ByVal VarPtr(OriginBytes(0)), 12
End Sub

Public Function Hook() As Boolean

Dim TmpBytes(0 To 11) As Byte
Dim p As LongPtr, osi As Byte
Dim OriginProtect As LongPtr

Hook = False

#If Win64 Then
    osi = 1
#Else
    osi = 0
#End If

pFunc = GetProcAddress(GetModuleHandleA("user32.dll"), "DialogBoxParamA")

If VirtualProtect(ByVal pFunc, 12, PAGE_EXECUTE_READWRITE, OriginProtect) <> 0 Then

    MoveMemory ByVal VarPtr(TmpBytes(0)), ByVal pFunc, osi + 1
    If TmpBytes(osi) <> &HB8 Then

        MoveMemory ByVal VarPtr(OriginBytes(0)), ByVal pFunc, 12

        p = GetPtr(AddressOf MyDialogBoxParam)

        If osi Then HookBytes(0) = &H48
        HookBytes(osi) = &HB8
        osi = osi + 1
        MoveMemory ByVal VarPtr(HookBytes(osi)), ByVal VarPtr(p), 4 * osi
        HookBytes(osi + 4 * osi) = &HFF
        HookBytes(osi + 4 * osi + 1) = &HE0

        MoveMemory ByVal pFunc, ByVal VarPtr(HookBytes(0)), 12
        Flag = True
        Hook = True
    End If
End If

End Function

Private Function MyDialogBoxParam(ByVal hInstance As LongPtr, _
    ByVal pTemplateName As LongPtr, ByVal hWndParent As LongPtr, _
    ByVal lpDialogFunc As LongPtr, ByVal dwInitParam As LongPtr) As Integer

If pTemplateName = 4070 Then
    MyDialogBoxParam = 1
Else
    RecoverBytes
    MyDialogBoxParam = DialogBoxParam(hInstance, pTemplateName, _
        hWndParent, lpDialogFunc, dwInitParam)
    Hook
End If

End Function


''''RUN THE CODE BELOW''''
Sub VBAUnprotected()

If Hook Then
    MsgBox "VBA Project is unprotected!", vbInformation, "*****"
End If

End Sub

```
---
## 🚀 How to Use

1. **Open** the Excel file you want to unprotect
2. Press **`Alt + F11`** to open the VBA Editor
3. Go to **Insert → Module**
4. **Paste** the macro code into the module
5. Press **`F5`** or click **Run**
6. Wait a moment — all sheets will be unprotected ✅
---
## ⚙️ Compatibility

| Environment | Supported |
|---|---|
| Microsoft Excel 2007+ | ✅ |
| Excel 365 | ✅ |
| `.xlsx` format | ✅ |
| `.xls` format | ✅ |
| Mac Excel | ⚠️ May vary |
| LibreOffice / Google Sheets | ❌ Not supported |

---

## ⚠️ Limitations

- Works on **sheet-level** protection only (not workbook-level password encryption)
- Does **not** bypass Excel's **Open Password** (file encryption) — that's a different, much stronger security layer
- May take a few seconds depending on system speed

---

## 📂 Repository Structure

```
📁 excel-sheet-unprotect/
│
├── 📄 README.md              ← You are here
└── 📄 Excel Sheet Password Remover.xlsb           ← Demo file with macro ready to run
```

---

## 🤝 Contributing

Contributions are welcome! Feel free to:
- Open an **Issue** for bugs or suggestions
- Submit a **Pull Request** with improvements
- Share it if it helped you 🙌

---

## 📜 License

This project is licensed under the **MIT License** — free to use, modify, and distribute.

---

## 👤 Author

Made with ❤️ by **ahmed samir**

[![GitHub](https://img.shields.io/badge/GitHub-Follow-black?logo=github)](https://github.com/ahmedsamir2010)

---

> 💡 **Tip:** If you found this useful, consider giving the repo a ⭐ — it helps others find it!
