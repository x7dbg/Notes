### 1DOS头

```c++
// DOS头结构体： _IMAE_DOS_HEADER
typedef struct _IMAE_DOS_HEADER {       //DOS .EXE header                 偏移  
    WORD e_magic;     //幻数  Magic number;                               0x00  
   	
    // 中间部分成员是为了兼容16位操作系统...可修改可忽略...
   
    LONG e_lfanew;     //File address of new exe header                   0x3C  
} IMAGE_DOS-HEADER, *PIMAGE_DOS_HEADER;  
```

### 2NT头

```c++
// NT头结构体： _IMAGE_NT_HEADERS
typedef struct _IMAGE_NT_HEADERS {
  DWORD                   Signature;		// 签名 32位文件格式的头部标识，不可修改 
  IMAGE_FILE_HEADER       FileHeader;		// 文件头
  IMAGE_OPTIONAL_HEADER32 OptionalHeader;	// 选项头
} IMAGE_NT_HEADERS32, *PIMAGE_NT_HEADERS32;

```

### 3文件头

```c++
// 文件头结构体 20B： _IMAGE_FILE_HEADER
typedef struct _IMAGE_FILE_HEADER {
  WORD  Machine; 		    	// 表示CPU平台,不可修改：
    							// 32位IMAGE_FILE_MACHINE_I386, 0x014c
    							// 64位IMAGE_FILE_MACHINE_AMD64, 0x8664
  WORD  NumberOfSections;    	// 表示节表数量，用于遍历节表，判断从PE中拷贝什么数据到内存中：
    							//.text/.rdata/.data...每个2行半
    							// 遍历节表经验：根据此处的个数拿对应的节表数据
  DWORD TimeDateStamp;			// 时间戳：链接器填写的文件生成的时间,作用不大(可修改)
  DWORD PointerToSymbolTable;   // 符号表位置(无用)
  DWORD NumberOfSymbols;	    // 符号表个数：windows的符号表信息一般由PDB放置在文件后端(无用)
  WORD  SizeOfOptionalHeader;   // 选项头大小：用于定位节表位置=选项头地址+选项头大小(不可随便修改)
  WORD  Characteristics;		// 文件属性，指应用程序是一个什么程序(不可随便修改)
} IMAGE_FILE_HEADER, *PIMAGE_FILE_HEADER;
```

```c++
typedef struct _IMAGE_OPTIONAL_HEADER {
  WORD  Magic;	// 32位PE： IMAGE_NT_OPTIONAL_HDR32_MAGIC  ,   0x10b. 
                // 以 _IMAGE_OPTIONAL_HEADER  结构体解析			
                // 64位PE： IMAGE_NT_OPTIONAL_HDR64_MAGIC  ,   0x20b.	
                // 以 _IMAGE_OPTIONAL_HEADER64  结构体解析			
  BYTE  MajorLinkerVersion;	// 主链接器版本号 (无用)
  BYTE  MinorLinkerVersion; // 副链接器版本号  (无用)
    
  //系统分配内存不看着3个值,但是对于调试器有影响(影响反汇编所用内存大小,OD是机器码个数*2,字节数是通过SizeOfCode 得到)   
  DWORD SizeOfCode;				// 代码所占空间大小  (没啥用)
  DWORD SizeOfInitializedData;	// 已初始化数据所占空间大小 (没啥用)
  DWORD SizeOfUninitializedData;// 未初始化数据所占空间大小 (没啥用)
    
  DWORD AddressOfEntryPoint;	// *oep:原本的程序入口点（实际为偏移，+模块基址=实际入口点）
    						    // ep: 被加工后的入口点
                                //这个值可以修改,但是修改过后必须跳转到在该偏移处跳转到真正入口

  DWORD BaseOfCode;	// 代码基址  (无用)
  DWORD BaseOfData;	// 数据基址   (无用)
    
  DWORD ImageBase;	// *建议模块地址：exe映射加载到内存中的首地址= PE 0处，即实例句柄hInstance
    			    // 一般而言，exe文件可遵从装载地址建议，但dll文件无法满足 (开了随机基址可能也不是这个值,是通过重定位表得到)
                    //这个值最好不要改,改的话要改动大量地方,因为函数和全局变量的地址也需要跟着改变
 
  DWORD SectionAlignment;   //内存对齐值,数据在内存的对齐值,很多内存地址,大小都要依赖他来计算
                            //默认1000h 一个分页大小,系统管理内存是以分页为单位
  DWORD FileAlignment;      //文件对齐值, 200h,磁盘的一个扇区大小  (vc6是1000h)
                            //文件的起始位置和大小都是跟文件对齐值对齐的 
                            // 对齐值都是2的倍数 如果把所有节合并了 就可以设为1,否则不可以随便修改因为要配合节检查
    
   //主副系统相关版本号   除了  MajorSubsystemVersion 不可修改,其他5个可以
  WORD  MajorOperatingSystemVersion;
  WORD  MinorOperatingSystemVersion;
  WORD  MajorImageVersion;
  WORD  MinorImageVersion;
  WORD  MajorSubsystemVersion;   //主子系统版本号   不可以修改  这里改成4可以再xp运行
  WORD  MinorSubsystemVersion;
    
    
  DWORD Win32VersionValue;   // win32版本值  xp上不可以改 ,win7和win10可以修改
    
  //改值是通过节表计算得到的   
  DWORD SizeOfImage;     //模块在内存中总大小,与 SectionAlignment 对齐,改的话不可以改变分页数量(但最好对齐)
  DWORD SizeOfHeaders;   // PE头总大小,与 FileAlignment 对齐
    
  DWORD CheckSum;       //校验值  3环程序随便改,0环程序会检查,不允许改 可以用 MapFileAndCheckSum 计算值
 
  WORD  Subsystem;      //子系统  不允许修改    /subsystem
    
  WORD  DllCharacteristics;   //描述应用程序的一些相关信息(例如是否开了随机基址等),可以改,但不能随便改
    
  //这四个值可以改,但是不能改得太离谱
  DWORD SizeOfStackReserve;  //栈保留
  DWORD SizeOfStackCommit;   //栈提交
  DWORD SizeOfHeapReserve;   //堆保留
  DWORD SizeOfHeapCommit;    //堆保留
    
    
  DWORD LoaderFlags;     //跟调试相关，目前用不到，值可以随便改
  DWORD NumberOfRvaAndSizes;   //下面数组个数（最小可以改为2,最大为16,前2个表是导入,导出表,必须要有）
 //数据目录表(成员2个 dword 第一个是内存偏移,第二个大小 )  
 //描述PE中各种个样的表的位置和大小,每个下标对应一个固定的表(前2个不能改,导入,导出表,改了无法调API)
  IMAGE_DATA_DIRECTORY DataDirectory[IMAGE_NUMBEROF_DIRECTORY_ENTRIES];  //柔性数组 个数由上面值决定,但是总大小为16个
} IMAGE_OPTIONAL_HEADER32, *PIMAGE_OPTIONAL_HEADER32;
```

### 5节表

```c++
// IMAGE_SECTION_HEADER 节表结构体，大小40B
typedef struct _IMAGE_SECTION_HEADER {
    
  BYTE  Name[IMAGE_SIZEOF_SHORT_NAME];	// 节表名称：描述性字段  2个字节
    
  // 下方4个字段：从文件S1处开始，拷贝S2大小的数据，到内存S3处，有效数据占用内存S4大小
  union {
    DWORD PhysicalAddress;
    DWORD VirtualSize;			// S4:内存大小
  } Misc;
  DWORD VirtualAddress;			// S3:内存地址：基于模块基址，与SectionAlignment对齐（0x1000）
  DWORD SizeOfRawData;			// S2:文件大小，与FileAlignment对齐（0x200）
  DWORD PointerToRawData;		// S1:文件偏移，与FileAlignment对齐（0x200）
 
  //跟调试相关  
  DWORD PointerToRelocations;	// 无用
  DWORD PointerToLinenumbers;	// 无用
  WORD  NumberOfRelocations;	// 无用
  WORD  NumberOfLinenumbers;	// 无用
    
  DWORD Characteristics;		// 节内存属性，取值IMAGE_SCN_...系列宏  分低位和高位
} IMAGE_SECTION_HEADER, *PIMAGE_SECTION_HEADER;
```

### 6导入表

```c++
typedef struct _IMAGE_IMPORT_DESCRIPTOR {
    
    //INT
    union {
        DWORD   Characteristics;    // 不是给Windows使用
       
        
        // 一个PIMAGE_THUNK_DATA结构体 保存 INT信息
        DWORD   OriginalFirstThunk; //一个API指向一个名称/序号表的中的表项，包括了序号和API名称两个字段  						    //可以理解为需要的API
    } DUMMYUNIONNAME;
    
    //下面2个字段没用
    DWORD   TimeDateStamp;          // 时间戳
    DWORD   ForwarderChain;         // -1 if no forwarders
    
    DWORD   Name;					// 重要，指向库名称
    
    //IAT
    DWORD   FirstThunk;             // IAT（导入地址表）,IMAGE_THUNK_DATA32,一样的格式，和OriginalFirstThunk
    							    //可以理解为API地址要填入的全局变量的地址
    							    //FirstThunk是该库需要的第一个API要填入的位置，
    							    //下一个偏移4的位置是第二个API要填入的位置
} IMAGE_IMPORT_DESCRIPTOR;
```

