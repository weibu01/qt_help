#### 1 共用体

```c
typedef union tagIntBuf
{
	int n;
	unsigned short nu16;
	unsigned char nu8;
	short n16;
	float f32;
	unsigned long nu32;
	double db;
	char buf[8];
}IntBuf;

可以用来实现类型转换   通过memcopy函数，，有时候强制类型转换会有问题。  比如下面代码 case 1 中，我原本想 char 直接强制转换为 unsigned char
结果我输入的值和打印出来的值对不上。。。

//获取 .dat 中的一条记录
//标准数据类型 0:s8,1:u8,2:s16,3:u16,4:s32,5:u32,6:float;7:string,8:double,9:long long;10:unsigned long long;11: BIT＿2：object;
static int getOneRecordPrintData(RDbLink *currdb,int recid,char * buf,int blen,short saveMode)  
{
	if(currdb && currdb->recbuf){
		ItemNode *itmp;
		IntBuf inb;
		int offset,size,i,len;
        char data[256];
		itmp = currdb->ilink->next;
		memset(buf,0,blen);
		for(i=0; i<currdb->itemnum && itmp; i++)
		{
			offset = currdb->reclen * recid + itmp->roff;
			size = itmp->size;
            if(size > 255)
                size = 255;
			memset(data,0,256);
            memcpy(&data[0],&currdb->recbuf[offset],size);   //一个字段的数据

			len = strlen(buf);
			switch(itmp->datatype){
				case 0:
					sprintf(&buf[len],"%d",(char*)data[0]);
					break;
				case 1:
					memcpy(&inb.nu8,data,1);
					sprintf(&buf[len],"%d",inb.nu8);   	
                    //sprintf(&buf[len],"%d",(unsigned char*)data[0]); 我之前用这种方式，结果我写入的值假如为 256，读出的值为负数。。。
                    //强制类型转换有问题
				break;
				case 2:
					memcpy(&inb.n16,data,2);
					if(saveMode != LITTLE_ENDIAN_MODE)
						endianSwap(&inb.n16,2);
					sprintf(&buf[len],"%d",inb.n16);  
					break;
				case 3:
					memcpy(&inb.nu16,data,2);
					if(saveMode != LITTLE_ENDIAN_MODE)
						endianSwap(&inb.nu16,3);
					sprintf(&buf[len],"%d",inb.nu16);
					break;
				case 4:
					memcpy(&inb.n,data,4);
					if(saveMode != LITTLE_ENDIAN_MODE)
						endianSwap(&inb.n,4);
					sprintf(&buf[len],"%ld",inb.n);
				case 5:
					memcpy(&inb.nu32,data,4);
					if(saveMode != LITTLE_ENDIAN_MODE)
						endianSwap(&inb.nu32,5);
					sprintf(&buf[len],"%ld",inb.nu32);
					break;
				case 6:
				{
					char str[RDB_MAX_NAME_LEN];
					memset(str,0,RDB_MAX_NAME_LEN);
					memcpy(&inb.f32,data,4);
					if(saveMode != LITTLE_ENDIAN_MODE)
						endianSwap(&inb.f32,6);
					_snprintf(str,RDB_MAX_NAME_LEN,"%f",inb.f32);
					removeSuffix(str);
					sprintf(&buf[len],"%s",str);  
					break;	
				}
				case 7:
					if(saveMode != LITTLE_ENDIAN_MODE)
						endianSwap(data,7);
					if((char*)data[0] == '\0')			//字符串为空时，显示空格
						sprintf(&buf[len],"%s","0");
					else
						sprintf(&buf[len],"%s",(char*)data);
					break;
				case 8:
				{
					char str[RDB_MAX_NAME_LEN];
					memset(str,0,RDB_MAX_NAME_LEN);
					memcpy(&inb.db,data,8);
					if(saveMode != LITTLE_ENDIAN_MODE)
						endianSwap(&inb.db,8);
					_snprintf(str,RDB_MAX_NAME_LEN,"%lf",inb.db); 
					removeSuffix(str);
					sprintf(&buf[len],"%s",str);
					break;		
				}
				default:
					break;
			}
			strcat(buf,",");
			itmp = itmp->next;
			if(itmp){
				if(strlen(buf)+itmp->size > blen)
					break;
			}
		}
		return 1;
	}
    return 0;
}
```



#### 2 结构体类型转换（子/大->基/小）

```c
typedef struct tagHMICaptionData
{
	unsigned short mode;
	unsigned short index;//播放到了多少帧；
	unsigned short speed;//多少秒播放完成；
	unsigned short pps;//每秒播放多少帧；
	HSDLBITMAP bitmap;
	char * caption;
}HMICaptionData;

typedef struct tagIndLightData
{
	HMICaptionData capdata;            
	short blinktime;//闪烁间隔；
	short blinkindex;//当前显示第几帧；
	short drawmode;//显示模式；
	short timermode;
}IndLightData;

IndLightData * inddata;
HMICaptionData * cdata = (HMICaptionData*)inddata; //类型转换  注意 HMICaptionData 在结构体 IndLightData 中处于首位才可以转换
```



#### 3 保留小数位数

```c
static double decimalConvert(double value,short pointAfterNum)
{
	int  num = 1;
	double ret;
	if(pointAfterNum <= 0 || value == (long long)value)
		return (long long)value;

	while (pointAfterNum > 0){
		num *= 10;
		--pointAfterNum;
	}

	ret = (long long)(num * value + 0.5) / (num*1.0);  // + 0.5 四舍五入
	return ret;
}
```



#### 4 atoi函数 空格

```c
#include <stdlib.h>
#include <stdio.h>

int main(void)
{
    int n;
    char *str = "99  ";
    n = atoi(str);
    printf("n=%d\n",n);
    return 0;
}

// str = “   99”   打印结果：99
// str = " "       打印结果：0
// str = "99    "  打印结果：99
// str = "9   9"   打印结果：9
说明atoi转换时会忽略首尾的空格  
```

