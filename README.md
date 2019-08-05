# WxJumpHelper

<br>

## 一、开发环境

<br>

1. 开发环境
   使用语言：C/C++
   IDE：VS2010+
2. 其他三方库
   EasyX(http://www.easyx.cn/downloads/)
   ADB(链接: https://pan.baidu.com/s/1nxrBXTB 密码: sfti)
3. ADB环境变量配置
   打开adb文件夹，将此路径添加到环境变量中；
   手机使用数据线连接电脑，并打开USB调试；
   打开cmd窗口，输入`adb devices`查看设备是否已连接，如图：
   ![tyt1](https://github.com/ycv587/WxJumpHelper/blob/master/images/tyt1.png)
4. EasyX安装
   双击打开安装文件，选择相应的VS版本即可，如图：
   ![tyt3](https://github.com/ycv587/WxJumpHelper/blob/master/images/tyt3.png)
5. 在VS中新建项目
   vs2017 - 文件 - 新建 - 项目 - Win32控制台应用程序 - 空项目 - 完成
   解决方案资源管理器 - 源文件 - 新建项 - 新建main.cpp

<br>

## 二、程序开发

<br>

### 2.1 使用adb操作手机

1. 使用adb命令截屏

   ```c++
   #include <stdio.h>
   #include <windows.h>
   #include <graphics.h> //EasyX库

   int main()
   {
   	// 截屏并保存
   	system("adb shell screencap -p /sdcard/screen.png");
   	system("adb pull /sdcard/screen.png");
   	// ...
   	return 0;
   }
   ```

   运行后打开项目路径检查是否存在screen.png，如图：
   ![tyt2](https://github.com/ycv587/WxJumpHelper/blob/master/images/tyt2.png)

2. 模拟长按屏幕

   ```c++
   // 在程序中添加以下代码：

   // 模拟手指按压
   // 按下坐标(200,300)，离开坐标(205,305)，按压时间500ms
   system("adb shell input swipe 200 300 205 305 500");
   ```

   打开微信跳一跳，运行程序，查看是否跳跃

<br>

### 2.2 处理截图 

1. 目的
   通过截图获取角色位置、待跳的方块位置、以及它们之间的距离，进而计算屏幕按压时间

2. 为了在程序中处理截图，需要将截图格式转化为jpg，添加部分代码后程序如下：

   ```c++
   #include <stdio.h>
   #include <windows.h>
   #include <graphics.h>
   #include "atlimage.h" // 处理图像格式

   IMAGE image; // 保存完整截图

   int main()
   {
   	initgraph(1080,1920); // 初始化窗口，与截图大小对应

   	system("adb shell screencap -p /sdcard/screen.png");
   	system("adb pull /sdcard/screen.png");
   	
   	CImage cimage;// 图像格式转换
   	cimage.Load(L"screen.png");
   	cimage.Save(L"screen.jpg");
   	
   	loadimage(&image, L"screen.jpg"); //将截图保存到image变量中
   	
   	putimage(0, 0, &image); //贴图
   	
   	// system("adb shell input swipe 200 300 205 305 500");
   	system("pause");
   	closegraph(); // 关闭图形界面
   	return 0;
   }
   ```

   运行程序，检查窗体是否成功加载截图，如图：
   ![tyt4](https://github.com/ycv587/WxJumpHelper/blob/master/images/tyt4.png)

3. 由于截图过大，且上下部分均不是游戏有效区域，故对图像进行裁剪，取得必须部分，经计算取得800*700的大小即可：

   ```c++
   #include <stdio.h>
   #include <windows.h>
   #include <graphics.h>
   #include "atlimage.h" // 处理图像格式

   IMAGE image,img; // 保存完整截图与裁剪后截图

   int main()
   {
   	initgraph(800,700); // 初始化窗口

   	system("adb shell screencap -p /sdcard/screen.png");
   	system("adb pull /sdcard/screen.png");
   	
   	CImage cimage; // 图像格式转换
   	cimage.Load(L"screen.png");
   	cimage.Save(L"screen.jpg");
   	
   	loadimage(&image, L"screen.jpg"); //将截图保存到image变量中
   	
   	SetWorkingImage(&image); // 修改工作区
   	getimage(&img, 100, 600, 800, 700); // 切割图片
   	SetWorkingImage(NULL); // 恢复工作区
   	
   	putimage(0, 0, &img); //贴图
   	
   	// system("adb shell input swipe 200 300 205 305 500");
   	system("pause");
   	closegraph(); // 关闭图形界面
   	return 0;
   }
   ```

   运行程序，如下：
   ![tyt5](https://github.com/ycv587/WxJumpHelper/blob/master/images/tyt5.png)

<br>

### 2.3 角色、方块定位

1. 在获取到游戏区域之后，便可以在此区域中通过颜色找到人物位置与方块位置，而像素代表了颜色，故需要取到每一个像素点：

   ```c++
   DWORD* pMem; //窗口显存
   int x,y;
   int xy[800][700]; // 保存img中所有的像素点

   pMem = GetImageBuffer(NULL); //获取窗口的显存

   SetWorkingImage(&img);
   for (y = 0; y < 700; y++)
   	for (x = 0; x < 800; x++)
   		xy[x][y] = getpixel(x, y); // 获取像素点

   // 贴图... 

   cleardevice(); // 清除屏幕
   // 重新贴图
   for (x = 0; x < 800; x++)
   {
   	for (y = 0; y < 700; y++)
   	{
   		int b = xy[x][y]&0xff; // 获取低8位
   		int g = (xy[x][y] >> 8) & 0xff; // 低8位去掉，再取低8位
   		int r = xy[x][y] >> 16; // 取高16位
   		pMem[y * 800 + x] = BGR(xy[x][y]);
   	}
   }
   FlushBatchDraw(); // 刷新缓存，显示图形
   ```

   以上代码可以获取到img中的每一个像素点，并且以像素点重新进行贴图

2. 在获取到所有点的颜色值之后，便可以开始根据颜色定位人物与跳跃点。从上到下开始逐像素扫描，则大部分情况下（极少数情况在后面处理）会先扫描到跳跃的方块，故以此定位跳跃的方块位置：

   ```c++
   int xx1,yy1; // 方块的顶部坐标

   SetWorkingImage(&img);
   for (y = 0; y < 700; y++)
   {
   	for (x = 0; x < 800; x++)
   	{
   		xy[x][y] = getpixel(x, y); // 获取像素点
       	// 自上而下进行扫描，若扫描到了方块的颜色，则跳出循环，不再获取像素点
       	if (isNewBlock(xy[x][y])) // 判断是否是新的盒子
       	{
         		xx1 = x;
         		yy1 = y;
         		goto next;
       	}
     	}
   }
   next: // ...
   
   // 判断是否是新的盒子
   BOOL isNewBlock(int color){}
   ```

3. 判断人物的位置，人物的颜色是固定的，故较容易取到。人物分为头和体，从上到下扫描则会扫描到头，那么就从下到上开始扫描：

   ```c++
   int peopleX,peopleY; // 人物坐标

   next:
   	for (y = 699; y >= 0; y--) // 自下而上开始扫描
   	{
   		for (x = 0; x < 800; x++)
   		{
   			xy[x][y] = getpixel(x, y); // 获取像素点
   			if (ColorFun(xy[x][y], RGB(55, 60, 100), 10)) //如果颜色是人物的颜色
   			{
   				peopleX = x; // 获得人物的坐标
   				peopleY = y;
   				break;
   			}
   		}
   		if (ColorFun(xy[x][y], RGB(55, 60, 100), 10))
   			break;
   	}


   // 判断颜色是否相似
   BOOL ColorFun(COLORREF color1, COLORREF color2, int diff){}
   ```

4. 在确定角色、方块的位置后，即可以开始计算距离并跳跃。但刚才所用到的ColorFun()与isNewBlock()还仍未提供实现细节。
   ColorFun()主要进行颜色相似的判断：

   ```c++
   BOOL ColorFun(COLORREF color1, COLORREF color2, int diff)
   {
   	// 取两种颜色的R、G、B值
   	int r1 = GetRValue(color1);
   	int g1 = GetGValue(color1);
   	int b1 = GetBValue(color1);

   	int r2 = GetRValue(color2);
   	int g2 = GetGValue(color2);
   	int b2 = GetBValue(color2);

   	if (sqrt(double((r2-r1)*(r2-r1) + (g2-g1)*(g2-g1) + (b2-b1)*(b2-b1))) < diff)
   	{
   		return TRUE;
   	}
   	return FALSE;
   }
   /*
   sqrt(double((r2-r1)*(r2-r1) + (g2-g1)*(g2-g1) + (b2-b1)*(b2-b1))
   颜色由R、G、B三基色组成，两个颜色值越接近，则其R、G、B值越接近
   三种基色的颜色差的平方根的值则代表两种颜色的相似度，值越小则越相似
   该函数的diff参数即为相似度，可以手动指定来确定颜色相似的程度
   */
   ```

   isNewBlock()收集了跳一跳中绝大多数盒子模型的颜色：

   ```c++
   BOOL isNewBlock(int color)
   {
   	// color为在img中取到的像素
   	int r = GetRValue(color);
   	int g = GetGValue(color);
   	int b = GetBValue(color);

   	if (colorFun(color, RGB(246, 246, 246), 10))//浅白色
   		return TRUE;
   	else if (colorFun(color, RGB(250, 250, 250), 10))
   		return TRUE;
   	else if (colorFun(color, RGB(255, 255, 255), 0))//纯白色
   		return TRUE;
   	else if (colorFun(color, RGB(100, 148, 106), 20))//墨绿色
   		return TRUE;
   	else if (colorFun(color, RGB(113, 113, 113), 10))//深灰色 
   		return TRUE;
   	else if (colorFun(color, RGB(245, 128, 58), 10))//橙色
   		return TRUE;
   	else if (colorFun(color, RGB(186, 239, 69), 10))//浅绿色
   		return TRUE;
   	else if (colorFun(color, RGB(234, 203, 174), 10))//木质桌子
   		return TRUE;
   	else if (colorFun(color, RGB(254, 240, 89), 10))//黄色
   		return TRUE;
   	else if (r > 124 && r < 134 && g>111 && g < 121 && b > 219 && b < 229)//紫色相间
   		return TRUE;
   	else if (r > 75 && r < 85 && g>158 && g < 165 && b > 85 && b < 95)//大圆绿柱子
   		return TRUE;
   	else if (colorFun(color, RGB(254, 74, 83), 10))//红色
   		return TRUE;
   	else if (colorFun(color, RGB(152, 114, 111), 10))//华岩石
   		return TRUE;
   	else if (colorFun(color, RGB(117, 117, 117), 10))//马桶
   		return TRUE;
   	else if (colorFun(color, RGB(225, 199, 142), 10))
   		return TRUE;
   	else if (colorFun(color, RGB(241, 241, 241), 10))//书本
   		return TRUE;
   	else if (colorFun(color, RGB(255, 172, 178), 10))//粉色盒子
   		return TRUE;
   	else if (colorFun(color, RGB(73, 73, 73), 3))//奶茶杯子
   		return TRUE;
   	else if (colorFun(color, RGB(147, 147, 147), 10))//类似唱片机
   		return TRUE;
   	return FALSE;
   }
   ```

5. 综合以上内容，加入代码：

   ```c++
   IMAGE image; //保存图片
   IMAGE img; //保存需要的图片，裁剪后
   int xy[800][700]; //保存img中所有的像素点
   DWORD* pMem; //窗口显存
   int x, y;
   int xx1, yy1; //第一个点的坐标，目标方块顶的坐标
   int peopleX, peopleY; //人的坐标

   BOOL isNewBlock(int color);
   BOOL ColorFun(COLORREF color1, COLORREF color2, int diff);

   int main()
   {
   	initgraph(800, 700); // 初始化窗口
   	pMem = GetImageBuffer(NULL); //获取窗口的显存

   	system("adb shell screencap -p /sdcard/screen.png");
   	system("adb pull /sdcard/screen.png");

   	CImage cimage;// 图像格式转换
   	cimage.Load(L"screen.png");
   	cimage.Save(L"screen.jpg");

   	loadimage(&image, L"screen.jpg"); //将截图保存到image变量中

   	SetWorkingImage(&image);// 修改工作区
   	getimage(&img, 100, 600, 800, 700);// 切割图片

   	SetWorkingImage(&img);
   	for (y = 0; y < 700; y++)
   	{
   		for (x = 0; x < 800; x++)
   		{
   			xy[x][y] = getpixel(x, y); // 获取像素点
   			if (isNewBlock(xy[x][y])) // 判断是否是新的盒子
   			{
   				xx1 = x;
   				yy1 = y;
   				goto next;
   			}
   		}
   	}
   next:
   	// 获取角色位置
   	for (y = 699; y >= 0; y--)
   	{
   		for (x = 0; x < 800; x++)
   		{
   			xy[x][y] = getpixel(x, y);
   			if (ColorFun(xy[x][y], RGB(55, 60, 100), 10))
   			{
   				peopleX = x;
   				peopleY = y;
   				break;
   			}
   		}
   		if (ColorFun(xy[x][y], RGB(55, 60, 100), 10))
   			break;
   	}

   	SetWorkingImage(NULL); // 恢复工作区
   	putimage(0, 0, &img); //贴图

   	getchar();

   	cleardevice(); // 清除屏幕

   	//重新贴图
   	for (x = 0; x < 800; x++)
   	{
   		for (y = 0; y < 700; y++)
   		{
   			int b = xy[x][y]&0xff; //获取低8位
   			int g = (xy[x][y] >> 8) & 0xff; //低8位去掉，再取低8位
   			int r = xy[x][y] >> 16; //取高16位

   			pMem[y * 800 + x] = BGR(xy[x][y]);
   		}
   	}
   	FlushBatchDraw(); //刷新缓存，显示图形

   	// system("adb shell input swipe 200 300 205 305 500");
   	system("pause");
   	closegraph(); // 关闭图形界面
   	return 0;
   }

   BOOL isNewBlock(int color){}
   BOOL ColorFun(COLORREF color1, COLORREF color2, int diff){}
   ```

   运行程序，如图：
   ![tyt6](https://github.com/ycv587/WxJumpHelper/blob/master/images/tyt6.png)
   角色自下而上进行扫描，方块自上而下进行扫描，在扫描到模型后停止扫描，故中间有一大块是无颜色填充的，这样就能更高效的定位人物和方块。

<br>

### 2.4 计算和跳跃

1. 在角色和方块定位时都获取到了坐标值，利用这个坐标值进行距离的计算，并根据距离与时间的关系计算长按屏幕的时间。

   ```c++
   // 计算目标跳点
   int xxx = xx1 + 10;
   int yyy = yy1 + 95;

   // 计算距离
   int distance = sqrt(double((yyy - peopleY)*(yyy - peopleY) + 
   	(xxx - peopleX)*(xxx - peopleX)));

   // 计算时间
   int time = 1.35 * distance;

   // 执行跳跃
   sprintf(str, "adb shell input swipe 200 300 205 305 %d",time);
   system(str);
   ```

2. 目标点的选定逻辑
   在定位时获取到了顶部坐标(xx1,yy1)，那么是否需要底部坐标，进而求中心点呢？这么做更精确，但是每次跳跃都会跳到中心，显然辅助被外挂检测机制检测到的概率更大，故选用下图所示逻辑计算跳点：
   ![tyt7](https://github.com/ycv587/WxJumpHelper/blob/master/images/tyt7.png)
   如图，以(x,y)作为方块顶点，各种模型的切点与切线均如图所示，以x和y增加部分偏移量而得出的目标点（绿线），对于小方块来说更靠近中间，但是大方块则基本不在中心，这样既考虑了小方块，又避免了外挂检测。**程序默认采用(x+10,y+95)进行跳点计算，可根据实际效果进行微调。**

3. 距离的计算
   距离采用勾股定理进行计算

4. 时间的计算
   **程序默认采用了1.35为系数进行时间计算，可根据实际效果进行微调。**

5. 此时程序如下：

   ```c++
   IMAGE image; //保存图片
   IMAGE img; //保存需要的图片，裁剪后
   int xy[800][700]; //保存img中所有的像素点
   DWORD* pMem; //窗口显存
   int x, y;
   int xx1, yy1; //第一个点的坐标，目标方块顶的坐标
   int peopleX, peopleY; //人的坐标

   BOOL isNewBlock(int color);
   BOOL ColorFun(COLORREF color1, COLORREF color2, int diff);

   int main()
   {
   	initgraph(800, 700); // 初始化窗口
   	pMem = GetImageBuffer(NULL); //获取窗口的显存

   	system("adb shell screencap -p /sdcard/screen.png");
   	system("adb pull /sdcard/screen.png");

   	CImage cimage;// 图像格式转换
   	cimage.Load(L"screen.png");
   	cimage.Save(L"screen.jpg");

   	loadimage(&image, L"screen.jpg"); //将截图保存到image变量中

   	SetWorkingImage(&image);// 修改工作区
   	getimage(&img, 100, 600, 800, 700);// 切割图片

   	SetWorkingImage(&img);
   	for (y = 0; y < 700; y++)
   	{
   		for (x = 0; x < 800; x++)
   		{
   			xy[x][y] = getpixel(x, y); // 获取像素点
   			if (isNewBlock(xy[x][y])) // 判断是否是新的盒子
   			{
   				xx1 = x;
   				yy1 = y;
   				goto next;
   			}
   		}
   	}
   next:
   	// 人物坐标
   	for (y = 699; y >= 0; y--)
   	{
   		for (x = 0; x < 800; x++)
   		{
   			xy[x][y] = getpixel(x, y);
   			if (ColorFun(xy[x][y], RGB(55, 60, 100), 10))
   			{
   				peopleX = x;
   				peopleY = y;
   				break;
   			}
   		}
   		if (ColorFun(xy[x][y], RGB(55, 60, 100), 10))
   			break;
   	}
	
   	SetWorkingImage(NULL); // 恢复工作区
   	putimage(0, 0, &img); //贴图
   	getchar();

   	cleardevice(); // 清除屏幕
   	//重新贴图
   	for (x = 0; x < 800; x++)
   	{
   		for (y = 0; y < 700; y++)
   		{
   			int b = xy[x][y]&0xff; //获取低8位
   			int g = (xy[x][y] >> 8) & 0xff; //低8位去掉，再取低8位
   			int r = xy[x][y] >> 16; //取高16位

   			pMem[y * 800 + x] = BGR(xy[x][y]);
   		}
   	}
   	FlushBatchDraw(); //刷新缓存，显示图形

   	// 计算目标点
   	int xxx = xx1 + 10;
   	int yyy = yy1 + 95;

   	int distance = sqrt(double((yyy-peopleY)*(yyy-peopleY)+(xxx-peopleX)*(xxx-peopleX)));
   	int time = 1.35 * distance;

   	sprintf(str, "adb shell input swipe 200 300 205 305 %d", time);
   	system(str);

   	system("pause");
   	closegraph(); // 关闭图形界面
   	return 0;
   }

   BOOL isNewBlock(int color){}
   BOOL ColorFun(COLORREF color1, COLORREF color2, int diff){}
   ```

<br>

### 2.5 优化改进

1. 在2.4完成之后，运行程序可以实现单步计算和跳跃，若要自动跳跃，则添加循环即可。

2. 关于图形界面
   在制作初期，图形界面可以更直观的显示计算与定位过程，但在程序开发结束后，图形显示部分可注释。

3. 极少数情况无法识别跳跃点
   在游戏中有一类极少遇到的情况，如下图：
   ![tyt8](https://github.com/ycv587/WxJumpHelper/blob/master/images/tyt8.png)
   如图，已跳过的方块的纵坐标小于待跳方块的纵坐标，所以会被先扫描到，对于这种情况，采用如下方案解决：先扫描角色，若角色x坐标小于400，则人物在左侧，那么人物左侧的像素就不必再被扫描，若人物在右侧，则人物右侧的内容也不必扫描。

4. 反检测
   触摸点：使用随机数选择触摸点，使操作更接近人。程序默认方案为：

   ```c++
   srand((unsigned int)time(NULL));
   // 使触摸点在一个小范围内随机选择
   touchX = rand() % 80 + 200; // 200-279
   touchY = rand() % 85 + 300; // 300-384
   ```

<br>

## 2.6 优化后代码

```c++
#include <stdio.h>
#include <windows.h>
#include <graphics.h>
#include "atlimage.h"
#include <time.h>

IMAGE image, img; // 截图图像
int coor[800][700]; // 保存截图（处理后）中所有的像素点
int blockX, blockY; // 目标方块顶部坐标
int peopleX, peopleY; // 角色的坐标
int touchX, touchY; // 模拟按下的坐标
int x, y;
char str[100];

BOOL isNewBlock(int color);
BOOL colorFun(COLORREF color1, COLORREF color2, int diff);

int main()
{
	srand((unsigned int)time(NULL));
	while (1)
	{
		// 截图并保存
		printf("capturing data and screen：");
		system("adb shell screencap -p /sdcard/screen.png");
		system("adb pull /sdcard/screen.png");
		// 截图格式转换 png -> jpg
		CImage cimage;
		cimage.Load(L"screen.png");
		cimage.Save(L"screen.jpg");
		loadimage(&image, L"screen.jpg"); // 把截图保存到image
		// 切割图片，取到游戏区，舍弃非游戏区域，加快判断效率
		SetWorkingImage(&image);
		getimage(&img, 100, 600, 800, 700);
		SetWorkingImage(&img);
		printf("getting the role and block location...\n");
		// 扫描角色坐标，从下往上（从上往下扫描到头部会停止）
		for (y = 699; y >= 0; y--)
		{
			for (x = 0; x < 800; x++)
			{
				coor[x][y] = getpixel(x, y);// 保存像素点
				if (colorFun(coor[x][y], RGB(55, 60, 100), 10))
				{
					peopleX = x;
					peopleY = y;
					goto getRole;
				}
			}
		}
	getRole:
		//  判断角色在左边还是右边，加快图形判断效率，处理极小形方块的bug
		for (y = 0; y < 700; y++)
		{
			for (x = ((peopleX < 400) ? peopleX + 60 : 0); x < ((peopleX < 400) ? 800 : peopleX - 60); x++)
			{
				coor[x][y] = getpixel(x, y); // 获取像素点
				if (isNewBlock(coor[x][y])) // 判断是否是新的盒子
				{
					blockX = x;
					blockY = y;
					goto getBlock;
				}
			}
		}
	getBlock:
		printf("calculate jump and touch coordinates...\n");
		// 计算目标点
		int targetX = blockX + 10;
		int targetY = blockY + 95;
		// 根据随机数种子模拟手指按下坐标，防外挂检测
		touchX = rand() % 80 + 200; // 200-279
		touchY = rand() % 85 + 300; // 300-384
		int distance = sqrt(double((targetY - peopleY)*(targetY - peopleY) + (targetX - peopleX)*(targetX - peopleX)));
		int time = 1.35 * distance;
		sprintf(str, "adb shell input swipe %d %d %d %d %d", touchX, touchY, touchX + 1, touchY + 1, time);
		printf("executing：(%d,%d)->(%d,%d) touching (%d,%d) for %dms\n", peopleX, peopleY, targetX, targetY, touchX, touchY, time);
		system(str);
		Sleep(1500);
	}
	return 0;
}

// 判断颜色是否相似，diff 越小越相似
BOOL colorFun(COLORREF color1, COLORREF color2, int diff)
{
	return sqrt(double((GetRValue(color2) - GetRValue(color1))*(GetRValue(color2) - GetRValue(color1)) + (GetGValue(color2) - GetGValue(color1))*(GetGValue(color2) - GetGValue(color1)) + (GetBValue(color2) - GetBValue(color1))*(GetBValue(color2) - GetBValue(color1)))) < diff;
}

// 判断是否是新的盒子
BOOL isNewBlock(int color)
{
	int r = GetRValue(color);
	int g = GetGValue(color);
	int b = GetBValue(color);

	if (colorFun(color, RGB(246, 246, 246), 10))//浅白色
		return TRUE;
	else if (colorFun(color, RGB(250, 250, 250), 10))
		return TRUE;
	else if (colorFun(color, RGB(255, 255, 255), 0))//纯白色
		return TRUE;
	else if (colorFun(color, RGB(100, 148, 106), 20))//墨绿色
		return TRUE;
	else if (colorFun(color, RGB(113, 113, 113), 10))//深灰色 
		return TRUE;
	else if (colorFun(color, RGB(245, 128, 58), 10))//橙色
		return TRUE;
	else if (colorFun(color, RGB(186, 239, 69), 10))//浅绿色
		return TRUE;
	else if (colorFun(color, RGB(234, 203, 174), 10))//木质桌子
		return TRUE;
	else if (colorFun(color, RGB(254, 240, 89), 10))//黄色
		return TRUE;
	else if (r > 124 && r < 134 && g>111 && g < 121 && b > 219 && b < 229)//紫色相间
		return TRUE;
	else if (r > 75 && r < 85 && g>158 && g < 165 && b > 85 && b < 95)//大圆绿柱子
		return TRUE;
	else if (colorFun(color, RGB(254, 74, 83), 10))//红色
		return TRUE;
	else if (colorFun(color, RGB(152, 114, 111), 10))//华岩石
		return TRUE;
	else if (colorFun(color, RGB(117, 117, 117), 10))//马桶
		return TRUE;
	else if (colorFun(color, RGB(225, 199, 142), 10))
		return TRUE;
	else if (colorFun(color, RGB(241, 241, 241), 10))//书本
		return TRUE;
	else if (colorFun(color, RGB(255, 172, 178), 10))//粉色盒子
		return TRUE;
	else if (colorFun(color, RGB(73, 73, 73), 3))//奶茶杯子
		return TRUE;
	else if (colorFun(color, RGB(147, 147, 147), 10))//类似唱片机
		return TRUE;
	return FALSE;
}
```

<br>

## 三、程序运行

<br>

### 3.1 程序运行

1. 安装adb
2. 配置adb环境变量
3. 手机使用数据线连接电脑，并打开USB调试
4. 打开cmd窗口，输入`adb devices`检测是否已连接
5. 手机打开微信跳一跳
6. 运行程序：如下图
   ![tyt9](https://github.com/ycv587/WxJumpHelper/blob/master/images/tyt9.png)

<br>

### 3.2 程序修改

1. 对于落地点的调整

   ```c++
   // 计算目标点
   int targetX = blockX + 10; // 修改 10
   int targetY = blockY + 95; // 修改 95
   ```

2. 对于距离与时间的关系微调

   ```c++
   int time = 1.35 * distance; // 修改 1.35
   ```

3. 对于新方块模型的扩充
   添加到isNewBlock()方法中即可
