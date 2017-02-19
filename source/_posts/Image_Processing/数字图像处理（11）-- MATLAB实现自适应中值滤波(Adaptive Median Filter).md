title: 数字图像处理（11）-- MATLAB实现自适应中值滤波(Adaptive Median Filter)
tags: 数字图像处理
categories: 数字图像处理
-----
在之前的图像降噪试验中，采取了一般的中值滤波方法，即选定窗口内的中间大小值作为输出像素的值，这在大部分情况都能反映窗口区域内像素的灰度情况。但是考虑到一种情况，假若窗口内分布着大量噪点，则选取的中值就有可能是噪声的灰度。为了避免这种情况，将在中值滤波过程中考虑对中值的判断及选取，这在Gonzalez的[《数字图像处理中》](http://item.jd.com/10658649.html)第五章称之为Adaptive Median Filter。

其业务流程如下：

###首先确定窗口内几个主要参数：

* $z_{min}$:窗口内最小灰度值（可能是salt）

* $z_{max}$:窗口内最大灰度值（可能是pepper）

* $z_{mid}$:窗口内中点灰度值

* $z_{xy}$:窗口中央元素灰度值

* $S_{max}$:最大允许窗口大小

###对窗口内中值的判断分为如下两步：

####StageA：

>$A_1$ = $z_{mid}$ - $z_{min}$

>$A_2$ = $z_{mid}$ - $z_{max}$

>IF $A_1$>0 AND $A_2$<0 go to stageB //当中值落在最大灰度与最小灰度间

>Else increase the window size //否则增加窗口大小重新选择中值直到合适

>If window size<=$S_max$ repeat StageA

>Else output $z_{mid}$

####StageB：

>$B_1$ = $z_{xy}$ - $z_{min}$

>$B_2$ = $z_{xy}$ - $z_{max}$

>//当窗口中央元素不为噪声时，/窗口中央元素的优先级高于中值

>If $B_1$>0 AND $B_2$<0 ,output $z_{xy}$

>Else output $z_{mid}$

具体实现代码如下:

```matlab
function [ imgDst ] = adaptiveMedianFilter( imgSrc,kernelSize, maxKernelSize )
%该函数完成自适应的中值滤波
    %params:    imgSrc:原图像
    %           kernelSize:卷积核大小（默认为3）
    %           maxKernelSize:卷积核大小（默认为3）
    
    %return:    imgDst:输出图像

    width = size(imgSrc,1);
    height = size(imgSrc,2);  
    
    %需要填充的边缘厚度
    numsPadding = floor(maxKernelSize/2);
    imgPadded = zeros([height+numsPadding*2,width+numsPadding*2]);
    imgPadded(numsPadding+1:numsPadding+height,numsPadding+1:numsPadding+width) = imgSrc;
    
    %下面是自适应的中值滤波的循环过程
    for i=numsPadding+1:numsPadding+height
        for j=numsPadding+1:numsPadding+width
            stageA( i,j,kernelSize,maxKernelSize,imgPadded );         
        end
    end
    
    %消去黑边
    imgDst = imgPadded(numsPadding+1:numsPadding+height,numsPadding+1:numsPadding+width);

end

function [ A1,A2,z,zMid,zMax,zMin ] = stageA( i,j,kernelSize,maxKernelSize,imgPadded )
%Stage A:
    while 1
        %获得窗口内像素点
        pixels = [];
        for k=-floor(kernelSize/2):floor(kernelSize)/2
            for l=-floor(kernelSize/2):floor(kernelSize)/2
                pixels = [pixels imgPadded(i+k,j+l)];
            end
        end
        len = length(pixels);
        zMax = max(pixels);
        zMin = min(pixels);
        zMid = median(pixels);
        z = pixels((len+1)/2); 
        A1 = zMid - zMin;
        A2 = zMid - zMax;
        if A1>0 && A2<0
            shouldBe = stageB(z,zMid,zMax,zMin);
            imgPadded(i,j) = zMid;
            break;
        else
            kernelSize = kernelSize + 2;
            if kernelSize > maxKernelSize
                 imgPadded(i,j) = zMid;
                 break;
            end
        end
    end
end

function [shouldBe] = stageB(z,zMid,zMax,zMin)
%Stage B:
    B1 = z-zMin;
    B2 = z-zMax;
    if B1>0 && B2<0
        shouldBe = z;
    else
        shouldBe = zMid;
    end   
end
```

测试代码如下：

```matlab
clear all;
close all;
imgPath = 'malight.bmp';
imgSrc = imread(imgPath);
%加入噪声
imgNoised = imnoise(imgSrc,'salt & pepper',0.02);
mKernelSize = 3;
imgDst1 = medianFilter(imgNoised,mKernelSize);
mKernelSize = 5;
imgDst2 = medianFilter(imgNoised,mKernelSize);
mkernelSize = 3;
maxKernelSize = 7;
tic
imgDst3 =  adaptiveMedianFilter( imgSrc,mKernelSize, maxKernelSize);
toc
%显示实验结果
figure('NumberTitle', 'off', 'Name', '中值滤波降噪');
subplot(221);
imshow(imgSrc);
title('原图像');
subplot(222);
imshow(imgNoised,[]);
title('噪声图像');
subplot(223);
imshow(imgDst1,[]);
title('卷积核大小为3的中值滤波');
subplot(224);
imshow(imgDst3,[]);
title('最大卷积核为7的自适应中值滤波');
```

执行效果：

![自适应中值滤波与一般中值滤波效果对比](http://7pulhb.com1.z0.glb.clouddn.com/ip-自适应中值滤波.jpg)

令人遗憾的是，虽然自适应中值滤波会带来更好的降噪效果，但是也会带来更长的时间开销，处理一张256*256大小的灰度图耗时达到了240~250秒左右(这有很大一部分原因是我的算法尚未优化所导致的)。
