//  main.cpp
//  parten
//
//  Created by 丛铭 on 2017/5/10.
//  Copyright © 2017年 丛铭. All rights reserved.
//

#include <iostream>
#include <stdio.h>
#include <stdlib.h>
#include "Config.h"
#include "PgmIO.h"
#include "RecognitionResult.h"
using namespace std;

int K_MAX=1;//调整，观察准确率的变化

int LoadTemplateImages(PgmImage **tmp,PgmImage **dst);
int TemplateMatching(PgmImage **tmp);
int Binarization(PgmImage **src, PgmImage **dst);//二值化

int main()
{

//--------------------------------------------------
    for(int i_MAX=3;i_MAX<4;i_MAX++) {
    K_MAX=i_MAX;
    PgmImage **tmp = new PgmImage*[TrainingSampleNum];
    PgmImage **dst = new PgmImage*[TrainingSampleNum];//存储二值化图像
    for (int i=0; i<TrainingSampleNum; i++){
        tmp[i] = new PgmImage(ImageSize, ImageSize);
        dst[i] = new PgmImage(ImageSize, ImageSize);
    }
    LoadTemplateImages(tmp,dst);

         TemplateMatching(tmp);

    for (int i=0; i<TrainingSampleNum; i++){
        delete  tmp[i];
        delete  dst[i];
    }
    delete[] tmp;
    delete[] dst;
}
    return 0;
}


long CalcL1Distance_m(PgmImage *img1, PgmImage *img2)//距离计算,改变算法看如何变化
{
    long dis = 0;
    long dis_m=0;
    int x, y;
    
    for  (y=0; y<img1->height; y++){
        for (x=0; x<img1->width; x++){
            dis = abs(img1->pixel[y][x] - img2->pixel[y][x]);
            //         cout<<"------------------------dis=:"<<dis<<endl;
            if (dis_m<dis) {
                dis_m=dis;
            }
        }
    }
    return dis_m;
}



long CalcL1Distance(PgmImage *img1, PgmImage *img2)//距离计算,改变算法看如何变化
{
    long dis = 0;
    int x, y;
    
    for  (y=0; y<img1->height; y++){
        for (x=0; x<img1->width; x++){
   //         cout<<"img1->pixel[y][x]"<<img1->pixel[y][x]-'0'+48<<endl;
  //          cout<<"img2->pixel[y][x]"<<img2->pixel[y][x]-'0'+48<<endl;
            //-------------------------------------------------------------真正的二值化，之前的把训练图想给二值化了
            
          
            int th_val=50;
            if (img1->pixel[y][x] < th_val)
                img1->pixel[y][x] = 0;
            else
                img1->pixel[y][x] = 255;
        
            
            //--------------------------------------------------------------
                dis += abs(img1->pixel[y][x] - img2->pixel[y][x])*abs(img1->pixel[y][x] - img2->pixel[y][x]);
   //         cout<<"------------------------dis=:"<<dis<<endl;
        }
    }
    return dis;
}

int Binarization(PgmImage **src, PgmImage **dst)//scr原图，dst处理图，修改阈值看如何变化,,错了不要管了
{
    char filename[256];
    int th_val = 180;//阈值
    int img_no=0;
    int img_dst=0;
    int each_class_num = TrainingSampleNum/ClassNum;
    for (int label=0; label<ClassNum; label++){
        for (int sample=0; sample<each_class_num; sample++){
            for (int y=0; y<src[img_dst]->height; y++){
                for (int x=0; x<src[img_dst]->width; x++){
            //        cout<<"img_dst=:"<<img_dst<<endl;
              //      cout<<"pixel[y][x]"<<src[img_dst]->pixel[y][x]<<endl;
                    if (src[img_dst]->pixel[y][x] < th_val)
                        dst[img_dst]->pixel[y][x] = 100;
                    else
                        dst[img_dst]->pixel[y][x] = 200;
                }
            }
            img_dst++;
            sprintf(filename, "/Users/congming/Desktop/Images/TrainingSamples/dstimages/binary_%d-%04d.pgm", label, sample);
            if (!SavePgmImage(filename, dst[img_no++]))
                return 0;
        }
    }
    return 1;
}


int LoadTemplateImages(PgmImage **tmp,PgmImage **dst)//读取画像
{
    char filename[256];
    
    int img_no = 0;
    int each_class_num = TrainingSampleNum/ClassNum;
    for (int label=0; label<ClassNum; label++){
        for (int sample=0; sample<each_class_num; sample++){
            sprintf(filename, TrainingDataFile, label, sample);
            printf("\rLoading the file %s\n", filename);
            if (!LoadPgmImage(filename, tmp[img_no++], label)){
                return 0;
            }
        }
    }
    Binarization(tmp, dst);
    return 1;
}

int ReturnMatchLabel(PgmImage *img, PgmImage **tmp)
{
   /* int match_label = -1;//----------------------------------step3
    int min_dis = INT_MAX;
    for (int i=0; i<TrainingSampleNum; i++){
        int dis = CalcL1Distance(img, tmp[i]);
        if (dis < min_dis){
            min_dis = dis;
            match_label = tmp[i]->label;
        }
    }*/
    //-----------------------------------------------------------step5
    int* k_label = new int[K_MAX];
    long* k_dist = new long[K_MAX];
    for (int k=0; k<K_MAX; k++) {
        k_label[k] = -1;
        k_dist[k] = INT_MAX;
    }
    for (int i=0; i<TrainingSampleNum; i++){
        long dis = CalcL1Distance_m(img, tmp[i]);//---------------------------------m原图像，不带m二值化
        
        for (int k=0; k<K_MAX; k++) {
            if (k_dist[k] > dis) {
                
                // k+1î‘ñ⁄Å`K_MAX-1î‘ñ⁄Ç‹Ç≈ÇÃóvëfÇÃçXêV
                for (int n=K_MAX-1; n>k; n--) {
                    k_label[n] = k_label[n-1];
                    k_dist[n] = k_dist[n-1];
                }
                // kî‘ñ⁄ÇÃóvëfÇÃçXêV
                k_label[k] = tmp[i]->label;
                k_dist[k] = dis;
                break;
            }
        }
    }
    
    int* class_cnt = new int[ClassNum];
    for (int i=0; i<ClassNum; i++)
        class_cnt[i] = 0;
    
    for (int i=0; i<K_MAX; i++)
        class_cnt[k_label[i]]++;
    
    int match_label = -1;
    int max_count = 0;
    for (int i=0; i<ClassNum; i++) {
        if (max_count < class_cnt[i]) {
            max_count = class_cnt[i];
            match_label = i;
        }
    }
    
    delete [] k_label;
    delete [] k_dist;
    delete [] class_cnt;
    

    //-------------------------------------------------------------
    return match_label;
}

int TemplateMatching(PgmImage **tmp)
{
    char filename[256];
    PgmImage *img = new PgmImage(ImageSize, ImageSize);
    RecognitionResult *result = new RecognitionResult(ClassNum);
    
    int each_class_num = TestSampleNum/ClassNum;	//
    int match_label = -1;     //实际匹配到的数字
    
    for (int label=0; label<ClassNum; label++){
        for (int sample=0; sample<each_class_num; sample++){
            sprintf(filename, TestDataFile, label, sample);
            printf("\rsearching the file %s\n", filename);
            if (!LoadPgmImage(filename, img, label)){
                return 0;
            }
            match_label = ReturnMatchLabel(img, tmp);	//img测试图像，tmp训练用图像
/*            if(sample==0)
            { cout<<"stop"<<endl;
                cout<<"match_lable=  "<<match_label<<endl;
            }*/
            result->res[img->label][match_label] ++;	//用于存储结果的二维数组
        }
    }
    
    result->CalcRatio();								//
    result->PrintRecognitionResult();					//
    sprintf(filename, RecognitionResultFile, TrainingSampleNum, TestSampleNum,K_MAX);
    result->SaveRecognitionResult(filename,K_MAX);			//
    
    cout<<"mark"<<endl;
    delete result;
    delete img;
    
    return 1;
}
