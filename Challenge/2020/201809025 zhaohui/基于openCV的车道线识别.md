# 基于openCV的车道线识别
## 实验目的即完成的功能
1.图像裁剪：通过设定图像ROI区域，拷贝图像获得裁剪图像

2.反透视变换：用拍摄的视频，没有对应的变换矩阵。所以建立二维坐标，通过四点映射的方法计算矩阵，进行反透视变化。后因ROI区域的设置易造成变换矩阵获取困难和插值像素得到的透视图效果不理想，故没应用。

3.二值化：先变化为灰度图，然后设定阈值直接变成二值化图像。

4.形态学滤波：对二值化图像进行腐蚀，去除噪点；然后对图像进行膨胀，弥补对车道线的腐蚀。

5.边缘检测：canny变化、sobel变化和laplacian变化中选择了效果比较好的canny变化，三者在代码中均可用，canny变化效果稍微好一点。

6.直线检测：实现了两种方法①使用opencv库封装好的霍夫直线检测函数，在原图对应区域用红线描出车道线②自己写了一种直线检测，在头文件中，遍历ROI区域进行特定角度范围的直线检测。 两种方法均可在视频中体现，第一种方法运行效率较快。

7.按键控制：空格暂停，其余键退出，方便调试和截图。
## 实验原理即所用的算法
### 边缘检测
> 灰度处理为图像处理的基本操作在这里不过多赘述。

#### 边缘检测的一般标准：
1. 以低的错误率检测边缘——尽可能准确的捕获图像中尽可能多的边缘。
2. 检测到的边缘应精确定位在真实边缘的中心。
3. 图像中给定的边缘应只被标记一次，并且在可能的情况下，图像的噪声不产生假的边缘。边缘只要一个精确的点宽度。
   
#### 边缘检测算法的处理流程
**1.高斯滤波 平滑图像**
> 为了尽可能减少噪声对边缘检测结果的影响，所l以必须滤除噪声以防止由噪声引起的错误检测。为了平滑图像，使用高斯滤波器与图像进行卷积，该步骤将平滑图像，以减少边缘检测器上明显的噪声影响。

**2.计算图像中每个像素点的梯度强度和方向**
**3.使用Non-Maximum Suppression(非极大值抑制)消除边缘检测带来的杂散响应**
**4.使用Double-Threshold(双阈值)检测确定真实和潜在的边缘**
**5.通过抑制鼓励的弱边缘最终完成边缘检测**
> 如果直接把梯度作为边缘的话，将得到一个粗边缘的图像，在图像边缘区域，其附近梯度值往往都很大，这不满足上面提到的准则(最小响应标准)，我们希望得到定位准确的单像素的边缘，所以将每个像素点的梯度与其梯度方向上的相邻像素比较，如果不是极大值，将其置0，.否则置为某.不大于255的数，非最大值抑制能帮助保留局部最大梯度而抑制所有其他梯度值。这意味着只保留了梯度变化中最锐利的位置。
## 实验结果即实现的效果
1.在亮度良好道路条件良好的情况下，检测车前区域的车道线实现比较成功，排除掉高速护栏的影响，而且原图像还能完整体现。
![11](11.png)
![12](12.png)

2.车子行驶在高速公路大型弯道上，可以在一定角度范围内认定车道线仍是直线，检测出为直线。
![21](21.png)
![22](22.png)

3.车子切换过程中只有一根车道线被识别，但是稳定回变换车道后，实现效果良好。减速线为黄色，二值化是也被过滤，没造成影响。
![31](31.png)
![32](32.png)

4.刚进入隧道时，摄像机光源基本处于高光状态，拍摄亮度基本不变，二值化图像时情况良好，噪声比较多但是没产生多大线状影响；当摄像头自动调节亮度，图像亮度变低，二值化时同一阈值把车道线给过滤掉，造成无法识别车道线的现象。
![41](41.png)
![42](42.png)
![51](51.png)
![52](52.png)

5.在道路损坏的情况下，由于阈值一定，基本上检测不出车道线。
![61](61.png)
![62](62.png)

## 实验结论
**实现的功能**：实现了车道线检测的基本功能，反透视变换矩阵实现了但效果不太理想，使用自己写的直线检测部分，车道线识别抗干扰能力较强。

**缺点**：整个识别系统都是固定的参数，只能在特定的环境产生良好的效果。

**改进空间**：提取全部关键参数，每次对ROI图像进行快速扫描更新参数，否则使用默认参数。例如，可以选择每次5间隔取点，以像素最高点的85%作为该次二值化的阈值。从而做到动态车道线识别。

## 完整代码
### 方法一：
### main.cpp
``` C++
 #include<cv.h>
 #include<cxcore.h>
 #include<highgui.h>
 include"mylinedetect.h"

 #include<cstdio>
 #include<iostream>
using namespace std;

int main(){
	//声明IplImage指针
	IplImage* pFrame = NULL;
	IplImage* pCutFrame = NULL;
	IplImage* pCutFrImg = NULL;
	//声明CvCapture指针
	CvCapture* pCapture = NULL;
	//声明CvMemStorage和CvSeg指针
	CvMemStorage* storage = cvCreateMemStorage();
	CvSeq* lines = NULL;
	//生成视频的结构
	VideoWriter writer("result.avi", CV_FOURCC('M', 'J', 'P', 'G'), 25.0, Size(856, 480));
	//当前帧数
	int nFrmNum = 0;
	//裁剪的天空高度
	int CutHeight = 310;
	//窗口命名
	cvNamedWindow("video", 1);
	cvNamedWindow("BWmode", 1);
	//调整窗口初始位置
	cvMoveWindow("video", 300, 0);
	cvMoveWindow("BWmode", 300, 520);
	//不能打开则退出
	if (!(pCapture = cvCaptureFromFile("lane.avi"))){
		fprintf(stderr, "Can not open video file\n");
		return -2;
	}
	//每次读取一桢的视频
	while (pFrame = cvQueryFrame(pCapture)){
		//设置ROI裁剪图像
		cvSetImageROI(pFrame, cvRect(0, CutHeight, pFrame->width, pFrame->height - CutHeight));
		nFrmNum++;
		//第一次要申请内存p
		if (nFrmNum == 1){
			pCutFrame = cvCreateImage(cvSize(pFrame->width, pFrame->height - CutHeight), pFrame->depth, pFrame->nChannels);
			cvCopy(pFrame, pCutFrame, 0);
			pCutFrImg = cvCreateImage(cvSize(pCutFrame->width, pCutFrame->height), IPL_DEPTH_8U, 1);
			//转化成单通道图像再处理
			cvCvtColor(pCutFrame, pCutFrImg, CV_BGR2GRAY);
		}
		else{
			//获得剪切图
			cvCopy(pFrame, pCutFrame, 0);
 #if 0		//反透视变换
			//二维坐标下的点，类型为浮点
			CvPoint2D32f srcTri[4], dstTri[4];
			CvMat* warp_mat = cvCreateMat(3, 3, CV_32FC1);
			//计算矩阵反射变换
			srcTri[0].x = 10;
			srcTri[0].y = 20;
			srcTri[1].x = pCutFrame->width - 5;
			srcTri[1].y = 0;
			srcTri[2].x = 0;
			srcTri[2].y = pCutFrame->height - 1;
			srcTri[3].x = pCutFrame->width - 1;
			srcTri[3].y = pCutFrame->height - 1;
			//改变目标图像大小
			dstTri[0].x = 0;
			dstTri[0].y = 0;
			dstTri[1].x = pCutFrImg->width - 1;
			dstTri[1].y = 0;
			dstTri[2].x = 0;
			dstTri[2].y = pCutFrImg->height - 1;
			dstTri[3].x = pCutFrImg->width - 1;
			dstTri[3].y = pCutFrImg->height - 1;
			//获得矩阵
			cvGetPerspectiveTransform(srcTri, dstTri, warp_mat);
			//反透视变换
			cvWarpPerspective(pCutFrame, pCutFrImg, warp_mat);
 #endif
			//前景图转换为灰度图
			cvCvtColor(pCutFrame, pCutFrImg, CV_BGR2GRAY);
			//二值化前景图
			cvThreshold(pCutFrImg, pCutFrImg, 80, 255.0, CV_THRESH_BINARY);
			//进行形态学滤波，去掉噪音
			cvErode(pCutFrImg, pCutFrImg, 0, 2);
			cvDilate(pCutFrImg, pCutFrImg, 0, 2);
			//canny变化
			cvCanny(pCutFrImg, pCutFrImg, 50, 120);
			//sobel变化
			//Mat pCutFrMat(pCutFrImg);
			//Sobel(pCutFrMat, pCutFrMat, pCutFrMat.depth(), 1, 1);
			//laplacian变化
			//Laplacian(pCutFrMat, pCutFrMat, pCutFrMat.depth());
 #if 1		//0为下面的代码，1为上面的代码
	#pragma region Hough直线检测
			lines = cvHoughLines2(pCutFrImg, storage, CV_HOUGH_PROBABILISTIC, 1, CV_PI / 180, 100, 15, 15);
			printf("Lines number: %d\n", lines->total);
			//画出直线
			for (int i = 0; i<lines->total; i++){
				CvPoint* line = (CvPoint*)cvGetSeqElem(lines, i);
				double k = ((line[0].y - line[1].y)*1.0 / (line[0].x - line[1].x));
				cout<<"nFrmNum "<<nFrmNum<<" 's k = "<<k<<endl;
				if(!(abs(k)<0.1))//去掉水平直线
					cvLine(pFrame, line[0], line[1], CV_RGB(255, 0, 0), 6, CV_AA);
			}
	#pragma endregion
 #else
	#pragma region mylinedetect
			Mat edge(pCutFrImg);
			vector<struct line> lines = detectLine(edge, 60);
			Mat pFrameMat(pFrame);
			drawLines(pFrameMat, lines);
			namedWindow("mylinedetect", 1);
			imshow("mylinedetect", pFrameMat);
	#pragma endregion
 #endif
			//恢复ROI区域
			cvResetImageROI(pFrame);
			//写入视频流
			writer << pFrame;
			//显示图像
			cvShowImage("video", pFrame);
			cvShowImage("BWmode", pCutFrImg);
			//按键事件，空格暂停，其他跳出循环
			int temp = cvWaitKey(2);
			if (temp == 32){
				while (cvWaitKey() == -1);
			}
			else if (temp >= 0){
				break;
			}
		}
	}
	//销毁窗口
	cvDestroyWindow("video");
	cvDestroyWindow("BWmode");
	//释放图像
	cvReleaseImage(&pCutFrImg);
	cvReleaseImage(&pCutFrame);
	cvReleaseCapture(&pCapture);

	return 0;
}

### mylinedetect.h
 #include "opencv2/imgproc/imgproc.hpp"
 #include "opencv2/highgui/highgui.hpp"
 #include <iostream>
 #include <vector>
 #include <cmath>
using namespace cv;
using namespace std;

const double pi = 3.1415926f;
const double RADIAN = 180.0 / pi;

struct line{
	int theta;
	int r;
};

vector<struct line> detectLine(Mat &img, int threshold){
	vector<struct line> lines;
	int diagonal = floor(sqrt(img.rows*img.rows + img.cols*img.cols));
	vector< vector<int> >p(360, vector<int>(diagonal));
	//统计数量
	for (int j = 0; j < img.rows; j++) {
		for (int i = 0; i < img.cols; i++) {
			if (img.at<unsigned char>(j, i) > 0){
				for (int theta = 0; theta < 360; theta++){
					int r = floor(i*cos(theta / RADIAN) + j*sin(theta / RADIAN));
					if (r < 0)
						continue;
					p[theta][r]++;
				}
			}
		}
	}
	//获得最大值
	for (int theta = 0; theta < 360; theta++){
		for (int r = 0; r < diagonal; r++){
			int thetaLeft = max(0, theta - 1);
			int thetaRight = min(359, theta + 1);
			int rLeft = max(0, r - 1);
			int rRight = min(diagonal - 1, r + 1);
			int tmp = p[theta][r];
			if (tmp > threshold
				&& tmp > p[thetaLeft][rLeft] && tmp > p[thetaLeft][r] && tmp > p[thetaLeft][rRight]
				&& tmp > p[theta][rLeft] && tmp > p[theta][rRight]
				&& tmp > p[thetaRight][rLeft] && tmp > p[thetaRight][r] && tmp > p[thetaRight][rRight]){
				struct line newline;
				newline.theta = theta;
				newline.r = r;
				lines.push_back(newline);
			}
		}
	}
	return lines;
}

void drawLines(Mat &img, const vector<struct line> &lines){
	for (int i = 0; i < lines.size(); i++){
		vector<Point> points;
		int theta = lines[i].theta;
		int r = lines[i].r;

		double ct = cos(theta / RADIAN);
		double st = sin(theta / RADIAN);

		//公式 r = x*ct + y*st
		//计算左边
		int y = int(r / st);
		if (y >= 0 && y < img.rows){
			Point p(0, y);
			points.push_back(p);
		}
		//计算右边
		y = int((r - ct*(img.cols - 1)) / st);
		if (y >= 0 && y < img.rows){
			Point p(img.cols - 1, y);
			points.push_back(p);
		}
		//计算上边
		int x = int(r / ct);
		if (x >= 0 && x < img.cols){
			Point p(x, 0);
			points.push_back(p);
		}
		//计算下边
		x = int((r - st*(img.rows - 1)) / ct);
		if (x >= 0 && x < img.cols){
			Point p(x, img.rows - 1);
			points.push_back(p);
		}
		//画线
		cv::line(img, points[0], points[1], Scalar(255, 0, 0), 5, CV_AA);
	}
}
```

### 方法二：
### namespace
```C++
 #include<cv.h>
 #include<cxcore.h>
 #include<highgui.h>

 #include<cstdio>
 #include<iostream>
using namespace std;

int main(){
	//声明IplImage指针
	IplImage* pFrame = NULL;
	IplImage* pCutFrame = NULL;
	IplImage* pCutFrImg = NULL;
	IplImage* pCutBkImg = NULL;
	//声明CvMat指针
	CvMat* pCutFrameMat = NULL;
	CvMat* pCutFrMat = NULL;
	CvMat* pCutBkMat = NULL;
	//声明CvCapture指针
	CvCapture* pCapture = NULL;
	//声明CvMemStorage和CvSeg指针
	CvMemStorage* storage = cvCreateMemStorage();
	CvSeq* lines = NULL;
	//当前帧数
	int nFrmNum = 0;
	//裁剪的天空高度
	int CutHeight = 250;
	//窗口命名
	cvNamedWindow("video", 1);
	//cvNamedWindow("background", 1);
	cvNamedWindow("foreground", 1);
	//调整窗口初始位置
	cvMoveWindow("video", 300, 30);
	cvMoveWindow("background", 100, 100);
	cvMoveWindow("foreground", 300, 370);
	//不能打开则退出
	if (!(pCapture = cvCaptureFromFile("lane.avi"))){
		fprintf(stderr, "Can not open video file\n");
		return -2;
	}
	//每次读取一桢的视频
	while (pFrame = cvQueryFrame(pCapture)){
		//设置ROI裁剪图像
		cvSetImageROI(pFrame, cvRect(0, CutHeight, pFrame->width, pFrame->height - CutHeight));
		nFrmNum++;
		//第一次要申请内存p
		if (nFrmNum == 1){
			pCutFrame = cvCreateImage(cvSize(pFrame->width, pFrame->height - CutHeight), pFrame->depth, pFrame->nChannels);
			cvCopy(pFrame, pCutFrame, 0);
			pCutBkImg = cvCreateImage(cvSize(pCutFrame->width, pCutFrame->height), IPL_DEPTH_8U, 1);
			pCutFrImg = cvCreateImage(cvSize(pCutFrame->width, pCutFrame->height), IPL_DEPTH_8U, 1);
			
			pCutBkMat = cvCreateMat(pCutFrame->height, pCutFrame->width, CV_32FC1);
			pCutFrMat = cvCreateMat(pCutFrame->height, pCutFrame->width, CV_32FC1);
			pCutFrameMat = cvCreateMat(pCutFrame->height, pCutFrame->width, CV_32FC1);
			//转化成单通道图像再处理
			cvCvtColor(pCutFrame, pCutBkImg, CV_BGR2GRAY);
			cvCvtColor(pCutFrame, pCutFrImg, CV_BGR2GRAY);
			//转换成矩阵
			cvConvert(pCutFrImg, pCutFrameMat);
			cvConvert(pCutFrImg, pCutFrMat);
			cvConvert(pCutFrImg, pCutBkMat);
		}
		else{
			//获得剪切图
			cvCopy(pFrame, pCutFrame, 0);
			//前景图转换为灰度图
			cvCvtColor(pCutFrame, pCutFrImg, CV_BGR2GRAY);
			cvConvert(pCutFrImg, pCutFrameMat);
			//高斯滤波先，以平滑图像
			cvSmooth(pCutFrameMat, pCutFrameMat, CV_GAUSSIAN, 3, 0, 0.0);
			//当前帧跟背景图相减
			cvAbsDiff(pCutFrameMat, pCutBkMat, pCutFrMat);
			//二值化前景图
			cvThreshold(pCutFrMat, pCutFrImg, 35, 255.0, CV_THRESH_BINARY);
			//进行形态学滤波，去掉噪音
			cvErode(pCutFrImg, pCutFrImg, 0, 1);
			cvDilate(pCutFrImg, pCutFrImg, 0, 1);
			//更新背景
			cvRunningAvg(pCutFrameMat, pCutBkMat, 0.003, 0);
			//pCutBkMat = cvCloneMat(pCutFrameMat);
			//将背景转化为图像格式，用以显示
			//cvConvert(pCutBkMat, pCutBkImg);
			cvCvtColor(pCutFrame, pCutBkImg, CV_BGR2GRAY);
			//canny变化
			cvCanny(pCutFrImg, pCutFrImg, 50, 100);
			#pragma region Hough检测
			lines = cvHoughLines2(pCutFrImg, storage, CV_HOUGH_PROBABILISTIC, 1, CV_PI / 180, 100, 30, 15);
			printf("Lines number: %d\n", lines->total);
			//画出直线
			for (int i = 0; i<lines->total; i++){
				CvPoint* line = (CvPoint* )cvGetSeqElem(lines, i);
				cvLine(pCutFrame, line[0], line[1], CV_RGB(255, 0, 0), 6, CV_AA);
			}
			#pragma endregion
			//显示图像
			cvShowImage("video", pCutFrame);
			cvShowImage("background", pCutBkImg);
			cvShowImage("foreground", pCutFrImg);
			//按键事件，空格暂停，其他跳出循环
			int temp = cvWaitKey(2);
			if (temp == 32){
				while (cvWaitKey() == -1);
			}
			else if (temp >= 0){
				break;
			}
		}
		//恢复ROI区域（多余可去掉）
		cvResetImageROI(pFrame);
	}
	//销毁窗口
	cvDestroyWindow("video");
	cvDestroyWindow("background");
	cvDestroyWindow("foreground");
	//释放图像和矩阵
	cvReleaseImage(&pCutFrImg);
	cvReleaseImage(&pCutBkImg);
	cvReleaseImage(&pCutFrame);
	cvReleaseMat(&pCutFrameMat);
	cvReleaseMat(&pCutFrMat);
	cvReleaseMat(&pCutBkMat);
	cvReleaseCapture(&pCapture);

	return 0;
}
```
#### 心得体会
1. 在前面几次的作业当中，我们已经明白了图像处理的一些基本原理及代码实现的操作方法，而此次设计报告是将我们已经学到的东西跟实际相结合，做出可以除了单张图片以外的图像变化。基于当前的智能驾驶的发展，对于图像处理的精度要求越来越高，做了这次简单的课程设计报告让我们学以致用，也慢慢体会到了图像处理的重要性。
2. 在做车道线检测之前我也尝试做过车牌检测，其中我觉得较困难的地方是调用信息与收集信息。也让我想到了路边拍摄违章车辆的摄像头能够从中提取出违章车辆也是用来车牌检测。
3. 我在这次课程设计报告做的过程中也使用过openVINO，但是由于在文件库里面下载文件不成功就此作罢。
> 在这里附上openVINO实验的一些入门知识网站。
> https://blog.csdn.net/u011385476/article/details/104769250
> openCV的简单介绍在前面的作业中有提到。
