```cpp
(DBC_Ctl *)(((DBM_ObjPage*)((unsigned)obj & DBM_PageMask))->GetCtl()); // 获得对象的 controller
```

```cpp
SYS_String Formate -> StdStringHelper::StringPrintf
SYS_String CompareNoCase -> StdStringHelper::CompareNoCase
SYS_String Empty() -> std::string clear();
SYS_String IsEmpty() -> std::string empty();
SYS_String LoadString(ID) -> std::string str; str = "ID";
SYS_String GetLength() = std::string length();

```

