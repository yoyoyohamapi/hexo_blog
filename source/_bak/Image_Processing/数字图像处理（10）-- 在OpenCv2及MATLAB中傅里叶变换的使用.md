title: 数字图像处理（10）-- 在OpenCv2及MATLAB中傅里叶变换的使用
tags: 数字图像处理
categories: 数字图像处理
--------
##opencv2中：

```cpp
#include <stdio.h>
#include <iostream>
#include <opencv2\imgproc\imgproc.hpp>
#include <opencv2\highgui\highgui.hpp>

using namespace cv;
using namespace std;

//Gloabla Variables
Mat imgSrc,imgFrq;
const char *winSrcName = "Image Src";
const char *winFrqName = "Image Frq scaled";


//主函数入口
int main(){
	//读取原图像,并转换为灰度图像
	const char *imgSrcPath = "E:\\Srcs\\Fig0323(a)(mars_moon_phobos).tif";
	imgSrc = imread(imgSrcPath,CV_LOAD_IMAGE_GRAYSCALE);
	
	if (imgSrc.empty())
		return -1;

	Mat padded;//扩展图像以加快离散傅里叶变换（当图像的尺寸是2， 3，5的整数倍时，计算速度最快）
	int m = getOptimalDFTSize(imgSrc.rows);//获得最佳变换尺寸
	int n = getOptimalDFTSize(imgSrc.cols);
	//扩展原图像至Padded图像中,边缘添加零,且边缘右下填充（0,m-imgSrc.rows代表顶部填充0行，底部填充m-imgSrc.rows行）
	copyMakeBorder(imgSrc,padded,0,m-imgSrc.rows,0,n-imgSrc.cols,BORDER_CONSTANT,Scalar::all(0));

	//为傅立叶变换的结果(实部和虚部)分配存储空间
	Mat planes[] = { Mat_<float>(padded), Mat::zeros(padded.size(), CV_32F) };
	Mat imgComplex;//新建复数域的图像
	merge(planes,2,imgComplex);//混合图像的实数部分和复数部分至新图像，亦即新图像每个像素点有实数，复数两个通道

	//执行傅里叶变换
	dft(imgComplex,imgComplex);
	//得到幅度值，去复数域
	split(imgComplex,planes);
	magnitude(planes[0], planes[1], planes[0]);//幅度 = （Real平方+Imginary平方）开根号
	Mat imgFrq = planes[0];//此时不标定，将会产生全白色图像（因为平方数很大）

	//对数化图像，增加亮度(输出图像 = log(1+输入图像))
	imgFrq += Scalar::all(1);
	log(imgFrq,imgFrq);
	/*
	*频谱图中心化
	*将频谱以中心划分为四个象限：
	*左上->1，左下->2，右下->3，右上->4
	*原象限与中心化后的象限对应关系：
	*1->3,  2->4,  3->1,  4->2
	*/
	int centerX = imgFrq.cols / 2;
	int centerY = imgFrq.rows / 2;
	Mat q1(imgFrq,Rect(0,0,centerX,centerY));
	Mat q2(imgFrq, Rect(0, centerY, centerX, centerY));
	Mat q3(imgFrq, Rect(centerX, centerY, centerX, centerY));
	Mat q4(imgFrq, Rect(centerX, 0, centerX, centerY));

	//执行交换
	Mat tmp;
	q1.copyTo(tmp);
	q3.copyTo(q1);
	tmp.copyTo(q3);

	q2.copyTo(tmp);
	q4.copyTo(q2);
	tmp.copyTo(q4);

	//标定
	normalize(imgFrq, imgFrq, 255.0,0);
	//显示原图像及其填充图像对应的傅里叶频谱
	namedWindow(winSrcName,CV_WINDOW_AUTOSIZE);
	imshow(winSrcName,imgSrc);
	namedWindow(winFrqName, CV_WINDOW_AUTOSIZE);
	imshow(winFrqName, imgFrq);
	waitKey(0);
}
```

执行结果如下：

![opencv2 傅里叶变换](http://7pulhb.com1.z0.glb.clouddn.com/ip-opencv2_傅里叶.png)

##Matlab中:

```matlab
imgSrc = imread('E:\\Srcs\\Fig0323(a)(mars_moon_phobos).tif');
%进行离散傅里叶变换
imgDst = fft2(imgSrc);
%中心化
imgDstCentered = fftshift(imgDst);
%得到幅度值
imgSpectrum = abs(imgDstCentered);
%对数增强
imgSpectrum = log(imgSpectrum);

%傅里叶反变换
imgIDFT = ifft2(imgDst);

%显示工作
figure;
subplot(131);
imshow(imgSrc);
subplot(132);
imshow(imgSpectrum,[]);
subplot(133);
imshow(imgIDFT,[]);
```

执行结果如下：

![matlab 傅里叶变换](http://7pulhb.com1.z0.glb.clouddn.com/ip-matlab_傅里叶.png)