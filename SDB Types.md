# SDB Types

定义了一些识别变量

定义了 SDBReadInfo 结构体

```cpp
struct SDBReadInfo {
	CArchive &m_archive;
	BOOL m_blSwapBytes;
	SYS_UShort m_usSDBRev;
	SYS_UShort m_dbRev;
}
```

