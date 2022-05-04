# STM32  SDCard  Use  SPI 

環境:

IDE:   IAR 8.20

Tool:  STM32CubeMX

HW:  NUCLEO-F401RE Develop board



## 1. Wiring



<img src="STM32  SDCard  Use  SPI.assets/1.png" alt="1"  />

## 2. STM32CubeMX Setting

### 選擇開發板 NUCLEO-F401RE

<img src="STM32  SDCard  Use  SPI.assets/2.png" alt="1"  />



<img src="STM32  SDCard  Use  SPI.assets/3.png" alt="1"  />

<img src="STM32  SDCard  Use  SPI.assets/4.png" alt="1"  />

### 開啟SWO功能

<img src="STM32  SDCard  Use  SPI.assets/5.png" alt="1"  />

### 設置 SPI2

預設時脈為84Mhz，為了後面方便Debug ， Clock 先除以 256 降頻處理

<img src="STM32  SDCard  Use  SPI.assets/6.png" alt="1"  />

### 設置 CS Pin 

調整上拉電阻，提高速度，設定別名 "SD_CS"  (方便Debug)

<img src="STM32  SDCard  Use  SPI.assets/7.png" alt="1"  />

### SPI2 訊號腳 設定上拉電阻

SCK、MISO、MOSI 皆需要設置 Pull-up 提升電阻

<img src="STM32  SDCard  Use  SPI.assets/8.png" alt="1"  />



### 開啟 FATFS 功能

勾選 "User-defined"

<img src="STM32  SDCard  Use  SPI.assets/9.png" alt="1"  />



### 調整 STACK Size

設定Stack為 0x1000 (4096 byte) ，使用FATFS  API 需要，不然會一直 Hard Fault Error !!

<img src="STM32  SDCard  Use  SPI.assets/10.png" alt="1"  />



## 3. IAR Project Setting



### 取消 Code Size 壓縮

Debug 時才不會一堆問題

<img src="STM32  SDCard  Use  SPI.assets/11.png" alt="1"  />

### 專案加入 SPI to FATFS 驅動檔案

將 "user_diskio_spi.c" 放進 Src 目錄 ，將 "user_diskio_spi.h" 放進 Inc目錄 

<img src="STM32  SDCard  Use  SPI.assets/12.png" alt="1"  />



<img src="STM32  SDCard  Use  SPI.assets/13.png" alt="1"  />



<img src="STM32  SDCard  Use  SPI.assets/14.png" alt="1"  />



<img src="STM32  SDCard  Use  SPI.assets/15.png" alt="1"  />



<img src="STM32  SDCard  Use  SPI.assets/16.png" alt="1"  />



## 4. 修改 Source Code

### main.h

讓 Driver 知道我使用的是 SPI2

```c
/* USER CODE BEGIN EFP */
#define SD_SPI_HANDLE hspi2                                                // == ADD THis ==
/* USER CODE END EFP */
```



使用 printf() & string 操作需要

```c
/* Private includes ----------------------------------------------------------*/
/* USER CODE BEGIN Includes */
#include "stdio.h"                         // == ADD THis ==
#include <string.h>                        // == ADD THis ==
/* USER CODE END Includes */
```



### user_diskio_spi.c

```c
#include "stm32f4xx_hal.h" /* Provide the low-level HAL functions */
#include "user_diskio_spi.h"
#include "main.h"                                                          // == ADD THis ==
//Make sure you set #define SD_SPI_HANDLE as some hspix in main.h
//Make sure you set #define SD_CS_GPIO_Port as some GPIO port in main.h
//Make sure you set #define SD_CS_Pin as some GPIO pin in main.h
extern SPI_HandleTypeDef SD_SPI_HANDLE;

/* Function prototypes */
```



先取消 SPI 速度變換，方便 Logic Analyzer 看 Cmd & Data 訊號

```c
inline DSTATUS USER_SPI_initialize (
                                    BYTE drv		/* Physical drive number (0) */
                                      )
{
  BYTE n, cmd, ty, ocr[4];
  
  if (drv != 0) return STA_NOINIT;		/* Supports only drive 0 */
  //assume SPI already init init_spi();	/* Initialize SPI */
  
  if (Stat & STA_NODISK) return Stat;	/* Is card existing in the soket? */
  
  //FCLK_SLOW();                                                                 // == Disable THis ==
  for (n = 10; n; n--) xchg_spi(0xFF);	/* Send 80 dummy clocks */
  
  ty = 0;
  if (send_cmd(CMD0, 0) == 1) {			/* Put the card SPI/Idle state */
    SPI_Timer_On(1000);					/* Initialization timeout = 1 sec */
    if (send_cmd(CMD8, 0x1AA) == 1) {	/* SDv2? */
      for (n = 0; n < 4; n++) ocr[n] = xchg_spi(0xFF);	/* Get 32 bit return value of R7 resp */
      if (ocr[2] == 0x01 && ocr[3] == 0xAA) {				/* Is the card supports vcc of 2.7-3.6V? */
        while (SPI_Timer_Status() && send_cmd(ACMD41, 1UL << 30)) ;	/* Wait for end of initialization with ACMD41(HCS) */
        if (SPI_Timer_Status() && send_cmd(CMD58, 0) == 0) {		/* Check CCS bit in the OCR */
          for (n = 0; n < 4; n++) ocr[n] = xchg_spi(0xFF);
          ty = (ocr[0] & 0x40) ? CT_SD2 | CT_BLOCK : CT_SD2;	/* Card id SDv2 */
        }
      }
    } else {	/* Not SDv2 card */
      if (send_cmd(ACMD41, 0) <= 1) 	{	/* SDv1 or MMC? */
        ty = CT_SD1; cmd = ACMD41;	/* SDv1 (ACMD41(0)) */
      } else {
        ty = CT_MMC; cmd = CMD1;	/* MMCv3 (CMD1(0)) */
      }
      while (SPI_Timer_Status() && send_cmd(cmd, 0)) ;		/* Wait for end of initialization */
      if (!SPI_Timer_Status() || send_cmd(CMD16, 512) != 0)	/* Set block length: 512 */
        ty = 0;
    }
  }
  CardType = ty;	/* Card type */
  despiselect();
  
  if (ty) {			/* OK */
    //FCLK_FAST();			/* Set fast clock */                               // == Disable THis ==
    Stat &= ~STA_NOINIT;	/* Clear STA_NOINIT flag */
  } else {			/* Failed */
    Stat = STA_NOINIT;
  }
  
  return Stat;
}

```



### user_diskio.c

```c
/* Includes ------------------------------------------------------------------*/
#include <string.h>
#include "ff_gen_drv.h"

/* Private typedef -----------------------------------------------------------*/
#include "user_diskio_spi.h"          // == ADD THis ==                                  
/* Private define ------------------------------------------------------------*/
```



#### User 初始化

```c
DSTATUS USER_initialize (
	BYTE pdrv           /* Physical drive nmuber to identify the drive */
)
{
  /* USER CODE BEGIN INIT */
  return USER_SPI_initialize (pdrv);               // == ADD THis ==       
  /* USER CODE END INIT */
}
```

#### User 狀態

```c
DSTATUS USER_status (
	BYTE pdrv       /* Physical drive number to identify the drive */
)
{
  /* USER CODE BEGIN STATUS */
  return USER_SPI_status (pdrv);                   // == ADD THis ==    
  /* USER CODE END STATUS */
}
```

#### User 讀取

```c
DRESULT USER_read (
	BYTE pdrv,      /* Physical drive nmuber to identify the drive */
	BYTE *buff,     /* Data buffer to store read data */
	DWORD sector,   /* Sector address in LBA */
	UINT count      /* Number of sectors to read */
)
{
  /* USER CODE BEGIN READ */
  return USER_SPI_read (pdrv, buff,  sector, count);    // == ADD THis ==
  /* USER CODE END READ */
}

```

#### User 寫入

```c
#if _USE_WRITE == 1
DRESULT USER_write (
	BYTE pdrv,          /* Physical drive nmuber to identify the drive */
	const BYTE *buff,   /* Data to be written */
	DWORD sector,       /* Sector address in LBA */
	UINT count          /* Number of sectors to write */
)
{ 
  /* USER CODE BEGIN WRITE */
  /* USER CODE HERE */
  return USER_SPI_write(pdrv, buff, sector, count);      // == ADD THis ==
  /* USER CODE END WRITE */
}
#endif /* _USE_WRITE == 1 */

```

#### User IO控制

```c
#if _USE_IOCTL == 1
DRESULT USER_ioctl (
	BYTE pdrv,      /* Physical drive nmuber (0..) */
	BYTE cmd,       /* Control code */
	void *buff      /* Buffer to send/receive control data */
)
{
  /* USER CODE BEGIN IOCTL */
  return USER_SPI_ioctl(pdrv, cmd, buff);
  /* USER CODE END IOCTL */
}
#endif /* _USE_IOCTL == 1 */
```



## 5. FATFS 功能測試



### a. Mount 

​        測試 printf() 功能是否正常，並且 Delay 1秒緩衝 讓 SD 卡 啟動。

```c
  printf("SD card SPI Demo\n");
  HAL_Delay(1000); //a short delay is important to let the SD card settle
```



```c
  //some variables for FatFs
  FATFS FatFs; 	// Fatfs handle
  FIL fil;      // File handle
  FRESULT fres; // Result after operations

  //Open the file system
  fres = f_mount(&FatFs, "", 1);  //1=mount now
  if (fres != FR_OK) {
    printf("f_mount error (%i)\r\n", fres);
    while(1);
  }else{
    printf("f_mount OK\n");
  }
```



```c
"FR_OK：成功",                               /* (0) Succeeded */
"FR_DISK_ERR：底層硬體錯誤",                  /* (1) A hard error occurred in the low level disk I/O layer */
"FR_INT_ERR：斷言失敗",                       /* (2) Assertion failed */
"FR_NOT_READY：物理驅動沒有工作",              /* (3) The physical drive cannot work */
"FR_NO_FILE：檔案不存在",                     /* (4) Could not find the file */
"FR_NO_PATH：路徑不存在",                     /* (5) Could not find the path */
"FR_INVALID_NAME：無效檔名",                  /* (6) The path name format is invalid */
"FR_DENIED：由於禁止訪問或者目錄已滿訪問被拒絕",  /* (7) Access denied due to prohibited access or
```

---



### b. GET  Size 取得大小

​          取得 SD 卡 Size 大小，可用空間大小。

```c
  
  DWORD free_clusters, free_sectors, total_sectors;
  
  FATFS* getFreeFs;
  
  fres = f_getfree("", &free_clusters, &getFreeFs);
  if (fres != FR_OK) {
    printf("f_getfree error (%i)\r\n", fres);
    while(1);
  }
  
  //Formula comes from ChaN's documentation
  total_sectors = (getFreeFs->n_fatent - 2) * getFreeFs->csize;
  free_sectors = free_clusters * getFreeFs->csize;
  
  printf("SD Card stats:\r\n%10lu KiB total drive space.\r\n%10lu KiB available.\r\n\n", total_sectors / 2, free_sectors / 2);
  
```



---

### c. WRITE 寫檔 



#### f_open

​        FA_WRITE | FA_OPEN_ALWAYS | FA_CREATE_ALWAYS   參數  Write 專用。

```c
  FRESULT Result;
  FIL file;      // File handle
  char FileName[] = "New.txt";
  
  Result = f_open( &file , FileName , FA_WRITE | FA_OPEN_ALWAYS | FA_CREATE_ALWAYS);

```



#### f_write

​       bytesWrote 用來確認實際寫了幾個 bytes

```c
  FRESULT Result;
  FIL file;      // File handle
  char FileName[] = "New.txt";
  
  Result = f_open( &file , FileName , FA_WRITE | FA_OPEN_ALWAYS | FA_CREATE_ALWAYS);

  if( Result == FR_OK ) {
    printf("I was able to open %s for writing\r\n" , FileName);
  } else {
    printf("f_open error (%i) \r\n", Result);
  }

  // Copy in a string
  char write_data_duffer[] = "This is a penguin!!";  
  unsigned int bytesWrote;  
  
  Result = f_write( &file , write_data_duffer ,sizeof(write_data_duffer) , &bytesWrote);
  
  if(Result == FR_OK) {
    printf("Write %i Bytes to %s !\r\n\n", bytesWrote , FileName);
  } else {
    printf("f_write error\r\n");
  }
  
  f_close(&file);
```



---

### e. READ 讀檔



#### f_open

​    FA_READ   參數  Read 專用。

```c
  FRESULT Result;
  FIL file;      // File handle
  char FileName[] = "New.txt";

　Result = f_open(&file, FileName , FA_READ);
```



#### f_gets

​      讀取固定數量。

```c
  FRESULT Result;
  FIL file;      // File handle
  char FileName[] = "New.txt";

　Result = f_open(&file, FileName , FA_READ);
  if (Result != FR_OK) {
    printf("f_open error\r\n");
    while(1);
  }
  printf("I was able to open %s for reading!\r\n\n" , FileName);
  
  //Read 30 bytes from "test.txt" on the SD card
  char readBuf[30];
  
  //We can either use f_read OR f_gets to get data out of files
  //f_gets is a wrapper on f_read that does some string formatting for us
  char* Rres = f_gets( readBuf , sizeof(readBuf) , &file);
  if(Rres != 0) {
    printf("f_gets String from %s contents: \r\n%s\r\n\n", FileName ,readBuf);
  } else {
    printf("f_gets error (%i) \r\n", *Rres);
  }

  f_close(&file);
```



#### f_lseek

重新定位索引。

```c
  Result = f_lseek ( &file, 0x00);	// back to start address
  if(Result == FR_OK){
    printf("f_lseek (0) ok\r\n\n");
  }else {
    printf("f_lseek error (%i) \r\n", Result );
  }
```



#### f_read

與 f_gets 差在可以知道實際讀了幾個 Bytes。

```c
  FRESULT Result;
  FIL file;      // File handle
  char FileName[] = "New.txt";
  char readBuffer[50];
  uint32_t bytesRead;

　Result = f_open(&file, FileName , FA_READ);
  if (Result != FR_OK) {
    printf("f_open error\r\n");
    while(1);
  }
  printf("I was able to open %s for reading!\r\n\n" , FileName);
  
  
  Result = f_read( &file , readBuffer , sizeof(readBuffer) , &bytesRead );

  if(Result == FR_OK){
    printf("Read (%d) bytes Data from %s contents: \r\n %s \r\n", bytesRead , FileName ,readBuffer);
  }else {
    printf("f_read error (%i) \r\n", Result );
  }
  
  f_close(&file);
```





---

## 6. SD 卡的暫存器



| Name |  Width (bit)  | Description                         |
| :--: | :-----------: | :---------------------------------- |
| CID  | 128 (16 Byte) | Card IDentification                 |
| RCA  |  16 (2 Byte)  | Relative Card  Address  (SDIO Only) |
| CSD  | 128 (16 Byte) | Card Specific Data                  |
| SCR  |  64 (8 Byte)  | SD card Configuration Register      |
| OCR  |  32 (4 Byte)  | Operating Conditions Register       |
| DSR  |  16 (2 Byte)  | Driver Stage Register               |
| SSR  | 512 (64 Byte) | SD Status Register                  |
| CSR  |  32 (4 Byte)  | Card Status Register                |



---

### CID 暫存器

CID 暫存器有 16 位元，它包含了本卡的識別碼（ID ）。這些資訊是在卡的生產期間被燒錄，Host 不能修改。

注意："SD" 卡和 "MMC" 卡 的 CID 暫存器結構上是不同的。

<img src="STM32  SDCard  Use  SPI.assets/17.png" alt="1"  />

---

### CSD 暫存器

CSD 有 128 位元，此暫存器包含了訪問該卡數據時的必要配置。

Cell type 欄內定義了 CSD 的各個區塊是 唯讀 R、可多次讀寫 R/W、只能一次寫入R/W（1）。

注意 "SD" 卡內的 CSD 暫存器和 "MMC"卡的結構不同。

在 SD3.0 協議中，CSD 分為版本 1.0 和版本 2.0，版本 1.0 對應 "標準容量" 的 SD 卡，版本 2.0 對應 "高容量" 和 "超高容量" 的 SD 卡。



<img src="STM32  SDCard  Use  SPI.assets/18.png" alt="1"  />





<img src="STM32  SDCard  Use  SPI.assets/19.png" alt="1"  />

---

### SCR 暫存器

SCR 提供了 SD 卡的一些特殊特屬性。長度是 64 位元。

製造商在生產廠時寫入    "MMC" 卡沒有 SCR。



DATA_STAT_AFTER_ERASE 定義了數據在抹除 (Erase) 後的狀態。是 "0" 或  "1" 其中一種（看供應商）。

SD_SECURITY  描述了該卡所支持的安全算法。 ( 0：無 ) 、( 1：安全協議1.0安全說明版本0.96) 、 ( 2：安全協議2.0 安全說明版本1.0 - 1.01)。其他 Reserved



<img src="STM32  SDCard  Use  SPI.assets/20.png" alt="1"  />

---

### OCR 暫存器

OCR  32 位元暫存器儲存了卡的VDD 電壓輪廓圖。任何標準的SD 卡主控制器可以使用2V 至3.6V 的工作電壓來讓SD 卡能執行這個電壓識別操作（CMD1）。

而訪問存儲器的陣列操作無論如何都需要 2.7V  至 3.6V  的工作電壓。OCR 暫存器顯示了在訪問卡的數據時所需要的電壓範圍



<img src="STM32  SDCard  Use  SPI.assets/21.png" alt="1"  />

---





## 7. SD 卡 SPI CMD 解析



每個命令的恆定長度為 6 個 Byte。
第一個位元組是 命令編 號 + 0x40 (64)的  EX: CMD8  =  0x40 + 0x8 = 0x48



### CMD List

| add Tx bit | CMD Index |     Parameter(v/x)     | Response | Data  Response | Description              | 說明                                                         |
| :--------: | :-------: | :--------------------: | :------: | :------------: | :----------------------- | ------------------------------------------------------------ |
|    0x40    |   CMD0    |           x            |    R1    |       x        | GO_IDLE_STATE            | 軟件重置                                                     |
|    0x41    |   CMD1    |           x            |    R1    |       x        | SEND_OP_COND             | 開啟初始化過程                                               |
|    0x69    |  ACMD41   |           2            | R1 (R3)  |       x        | APP_SEND_OP_COND         | 僅SD卡。開啟初始化過程                                       |
|    0x48    |   CMD8    |           3            |    R7    |       x        | SEND_IF_COND             | 僅SD卡V2。檢查電壓範圍                                       |
|    0x49    |   CMD9    |           x            |    R1    |       v        | SEND_CSD                 | 讀取CSD寄存器                                                |
|    0x4A    |   CMD10   |           x            |    R1    |       v        | SEND_CID                 | 讀取CID寄存器                                                |
|    0x4C    |   CMD12   |           x            |   R1b    |       x        | STOP_TRANSMISSION        | 停止讀取數據                                                 |
|    0x50    |   CMD16   |    Block Len (31:0)    |    R1    |       x        | SET_BLOCKLEN             | 設置 Block 讀寫操作的長度，SDHC/SDXC card默認值都是 512 Bytes。 |
|    0x51    |   CMD17   |      Addr (31:0)       |    R1    |       v        | READ_SINGLE_BLOCK        | 讀一個 Block                                                 |
|    0x52    |   CMD18   |      Addr (31:0)       |    R1    |       v        | READ_MULTIPLE_BLOCK      | 讀多個 Block                                                 |
|    0x57    |   CMD23   | Number of block (15:0) |    R1    |       x        | SET_BLOCK_COUNT          | 僅MMC。定義下一個多 Block 讀寫命令要傳輸的 Block 個數        |
|    0x57    |  ACMD23   | Number of block (22:0) |    R1    |       x        | SET_WR_BLOCK_ERASE_COUNT | 僅SD卡。定義下一個多 Block 寫入命令要預擦除的 Block 個數     |
|    0x58    |   CMD24   |      Addr (31:0)       |    R1    |       v        | WRITE_BLOCK              | 寫一個 Block                                                 |
|    0x59    |   CMD25   |      Addr (31:0)       |    R1    |       v        | WRITE_MULTIPLE_BLOCK     | 寫多個 Block.                                                |
|    0x77    |   CMD55   |           x            |    R1    |       x        | APP_CMD                  | ACMD<n>命令的開頭命令                                        |
|    0x7A    |   CMD58   |           x            |    R3    |       x        | READ_OCR                 | 讀OCR寄存器                                                  |



---

### CMD8 解析

​	CMD8（SEND_IF_COND），帶入參數VHS（告知 SD卡 Host 支援的電壓範圍）， Check Pattern 可隨意輸入（官方建議0xAA）。

#### Send :

<img src="STM32  SDCard  Use  SPI.assets/22.png" alt="1"  />

##### VHS 定義

​	告知SD卡系統提供的電壓範圍

<img src="STM32  SDCard  Use  SPI.assets/23.png" alt="1"  />

##### Check Patten 

​	Check Patten 官方建議使用 0xAA



##### CRC7 

​	CRC7 的計算，大部分硬體 SPI 會幫忙做掉，不需要自己計算。

​	詳細演算法參考: https://stackoverflow.com/questions/49672644/cant-figure-out-how-to-calculate-crc7

```c
// 此算法已驗證過是正確的 !!
// 編譯 $ gcc -o app main.c
// 執行 $ ./app

#include "stdio.h"
#include "string.h"

unsigned char CRC7(const unsigned char message[], const unsigned int length) {
  const unsigned char poly = 0b10001001;
  unsigned char crc = 0;
  for (unsigned i = 0; i < length; i++) {
     crc ^= message[i];
     for (int j = 0; j < 8; j++) {
      // crc = crc & 0x1 ? (crc >> 1) ^ poly : crc >> 1;       
      crc = (crc & 0x80u) ? ((crc << 1) ^ (poly << 1)) : (crc << 1);
    }
  }
  //return crc;
  return crc >> 1;
}

int main(){

    unsigned char crc7_out;
    unsigned char crc_sd_card;

    printf("start\n");
    const unsigned char rawdata[] = {0x48,0x00,0x00,0x01,0xAA};

    crc7_out = CRC7( rawdata , 5 );

    crc_sd_card = (crc7_out << 1) + 1; // crc7_out  [7 bit << 1]  +  (b'00000001) [SPI End Patten]

    printf("0x%X\n", crc_sd_card);

    return 0;
}
```

---



#### Response :

​	CMD8   Response  Format:   R7

<img src="STM32  SDCard  Use  SPI.assets/24.png" alt="1"  />



##### R7 解析

​	基本上 Host端發送 CMD8 ， SD卡就回覆 Comand Version  、  VCA ( VHS )  、  Check Patten ( Host傳什麼 SD卡就回什麼 )。



---

##### R1、R3定義

<img src="STM32  SDCard  Use  SPI.assets/25.png" alt="1"  />

##### R7 定義

<img src="STM32  SDCard  Use  SPI.assets/26.png" alt="1"  />



---

### ACMD41 解析



#### Send :

​			ACMD41（APP_SEND_OP_COND），帶入的參數 HCS（告訴 SD 卡 ，Host 是否支援高速通訊）。

<img src="STM32  SDCard  Use  SPI.assets/27.png" alt="1"  />

<img src="STM32  SDCard  Use  SPI.assets/28.png" alt="1"  />



對於支援 CMD8 的卡，Host 設定 ACMD41 的引數 HCS=1，告訴SD卡，Host 支援 SDHC卡。

對 2.0 的卡，OCR 的 CCS 位元用於表示 SDHC 還是 SDSC ； 

對 1.x 的卡，則忽略該位元；

對 MMC 卡，則不支援 ACMD41，MMC 卡只需要傳送：CMD0 和 CMD1 即可完成初始化。



##### OCR 定義

<img src="STM32  SDCard  Use  SPI.assets/29.png" alt="1"  />

---



#### Response :

​	ACMD41   Response  Format:   R1

##### R1 解析

<img src="STM32  SDCard  Use  SPI.assets/30.png" alt="1"  />

In Idle State = 0  代表 SD 初始化完成

<img src="STM32  SDCard  Use  SPI.assets/31.png" alt="1"  />



---

### CMD58 解析



#### Send :

​		CMD58 （READ_OCR ），不需帶入參數，如果沒有特別開啟就不再需要 CRC7。

<img src="STM32  SDCard  Use  SPI.assets/32.png" alt="1"  />



---

#### Response :



​	CMD58   Response  Format :  R3，SD卡回覆：是否完成電源初始化、是否支援高速通訊、支援電壓範圍、

<img src="STM32  SDCard  Use  SPI.assets/33.png" alt="1"  />





##### R1 解析

​	In Idle State = 0 表示 代表 SD 初始化完成

<img src="STM32  SDCard  Use  SPI.assets/34.png" alt="1"  />

##### OCR 解析

​	Card Power up status = 1 表示 SD卡已完成電源初始化。

​	CCS = 1 表示此卡支援高速傳輸。

​	( 15 : 23 ) = 1 表示 VDD 電壓支援範圍 2.7 ~ 3.6V。

<img src="STM32  SDCard  Use  SPI.assets/35.png" alt="1"  />



---

### CMD17 解析



#### Send :

​		CMD17 （READ_SINGLE_BLOCK），讀取 SD卡一個 Block 資料，預設 512 Byte，參數為 4 個 byte Address，如果沒有特別開啟就不再需要 CRC7。

<img src="STM32  SDCard  Use  SPI.assets/36.png" alt="1"  />

---

#### Response :

​	CMD17  Response  512 Bytes  Memory  Data  from  parameter "memory address".



等待 SD 卡傳送 0xFE

Memory 資料 512 位元組

末尾2位元為CRC

<img src="STM32  SDCard  Use  SPI.assets/37.png" alt="1"  />



---

### CMD24 解析

#### Send & Response

​		CMD24 （WRITE_BLOCK），寫入 SD卡一個 Block 資料，預設 512 Bytes，參數為 4 個 byte Address，如果沒有特別開啟就不再需要 CRC7。

​		 SD 回應 R1 ，Host 發送 Start Pattern ( 0xFE ) 後即可接續發送 512 Bytes Data，結尾串 2 Byte CRC ( 固定0xFF 0xFF ) 。

​		 傳送完成，SD卡會回 b'(xxx00101)       EX: xxx = b'(111)     Final Byte = 0xE5。		 



<img src="STM32  SDCard  Use  SPI.assets/38.png" alt="1"  />





當 SD卡 把 512 Bytes 寫進 Flash 的期間 MISO 會被拉 Low，待資料寫完畢後 MISO 拉 High ，Host 才可以傳送 CMD

<img src="STM32  SDCard  Use  SPI.assets/39.png" alt="1"  />