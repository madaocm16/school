//  main.cpp
//  parten
//
//  Created by 丛铭 on 2017/5/10.
//  Copyright © 2017年 丛铭. All rights reserved.
//

#include <iostream>
#include <stdio.h>
#include <stdlib.h>

#define        TrainingSampleNum    2000
#define        TestSampleNum        10000
#define        ClassNum            10
#define        ImageSize            28

using namespace std;
int K_MAX=1;//調整、正確率がわかる
int LoadTemplateImages(PgmImage **tmp,PgmImage **dst);
int TemplateMatching(PgmImage **tmp);
int Binarization(PgmImage **src, PgmImage **dst);//二值化
const char TrainingDataFile[] = "/Users/congming/Desktop/Images/TrainingSamples/%d-%04d.pgm";
const char TestDataFile[] = "/Users/congming/Desktop/Images/TestSamples/%d-%04d.pgm";
const char RecognitionResultFile[] = "/Users/congming/Desktop/パターン認識特論/parten/result-training_%05d-test_%05d_K_MAX=%d.txt";

int main()
{
    int i,i_MAX;
    
    for(i_MAX=3;i_MAX<4;i_MAX++) {
    K_MAX=i_MAX;
    PgmImage **tmp = new PgmImage*[TrainingSampleNum];
    PgmImage **dst = new PgmImage*[TrainingSampleNum];//二值化の図を保存
        
    for (i=0; i<TrainingSampleNum; i++){
        tmp[i] = new PgmImage(ImageSize, ImageSize);
        dst[i] = new PgmImage(ImageSize, ImageSize);
    }
    LoadTemplateImages(tmp,dst);
    TemplateMatching(tmp);
        
    for (i=0; i<TrainingSampleNum; i++){
        delete  tmp[i];
        delete  dst[i];
    }
    delete[] tmp;
    delete[] dst;
}
    return 0;
}

/*認識結果を保存する構造体や認識率を算出する関数*/
typedef struct RecognitionResult {
    int class_num;
    int **res;
    double *ratio;
    double average_ratio;
    double variance;
    
    RecognitionResult(int _class_num)
    {
        int i, j;
        class_num = _class_num;
        res = new int*[class_num];
        
        for (i=0; i<class_num; i++){
            res[i] = new int [class_num];
        }
        
        for (i=0; i<class_num; i++){
            
            for (j=0; j<class_num; j++){
                res[i][j] = 0;
            }
        }
        ratio = new double [class_num];
        
        for (i=0; i<class_num; i++){
            ratio[i] = 0.0;
        }
        average_ratio = 0.0;
    }
    
    ~RecognitionResult()
    
    {
        int i;
        
        for (i=0; i<class_num; i++){
            delete [] res[i];
        }
        delete [] res;
    }
    
    void CalcRatio()　//認出率の計算
    {
        int i, j;
        int correct, total;
        
        for (i=0; i<class_num; i++){
            total = 0;
            
            for (j=0; j<class_num; j++){
                total += res[i][j];
                if (i==j){
                    correct = res[i][j];
                }
            }
            ratio[i] = (double)correct/total;
            average_ratio += ratio[i];
        }
        average_ratio /= class_num;
    }
    
    void PrintRecognitionResult()　　//結果を出力
    {
        int i, j;
        printf("\n\n");
        
        for (i=0; i<class_num; i++){
            printf("\tC%d", i);
        }
        printf("\n");
        
        for (i=0; i<class_num; i++){
            printf("C%d", i);
            
            for (j=0; j<class_num; j++){
                printf("\t%4d", res[i][j]);
            }
            printf("\n");
        }
        printf("\nRecognition Ratio\n");
        
        for (i=0; i<class_num; i++){
            printf("Class %d\t%3.2lf\n", i, ratio[i]*100);
        }
        printf("Average\t%2.2lf\n",average_ratio*100);
    }
    
    void SaveRecognitionResult(char *filename,int K_MAX) //結果を保存
    {
        int i, j;
        FILE *fp;
        fp=fopen(filename, "w");
        if(fopen( filename, "w" ) == NULL){
            printf("The file \"%s\" was not opened.", filename);
            return;
        }
        
        for (i=0; i<class_num; i++){
            fprintf(fp, "\tC%d", i);
        }
        fprintf(fp, "\n");
        
        for (i=0; i<class_num; i++){
            fprintf(fp, "C%d", i);
            
            for (j=0; j<class_num; j++){
                fprintf(fp, "\t%4d", res[i][j]);
            }
            fprintf(fp, "\n");
        }
        fprintf(fp, "\nRecognition Ratio\n");
        
        for (i=0; i<class_num; i++){
            fprintf(fp, "Class %d\t%3.2lf\n", i, ratio[i]*100);
        }
        fprintf(fp, "Average\t%2.2lf\n",average_ratio*100);
        fprintf(fp, "K_Max\t%d\n",K_MAX);
        variance=0;
        
        for (i=0; i<class_num; i++)
        {
            variance+=(1.00-ratio[i])*(1.00-ratio[i])*10000;
        }
        variance=variance/class_num;
        fprintf(fp, "Variance\t%2.2lf\n",variance);
        fclose(fp);
    }
}

/*画像の扱いを簡単にするための実装*/
/*画像構造体と画像を入出力する関数*/
typedef struct PgmImage {
    int width;
    int height;
    int label;  //画像のクラスラベル
    unsigned char **pixel; //画像を格納するメモリへのポインタ

    PgmImage(int _width, int _height)
    {
        int y;
        width = _width;
        height = _height;
        label = -1;
        pixel = new unsigned char*[height];
        
        for (y=0; y<height; y++){
            pixel[y] = new unsigned char[width];
        }
    }
    
    ~PgmImage()
    
    {
        int y;
        
        for (y=0; y<height; y++){
            delete [] pixel[y];
        }
        delete [] pixel;
    }
} PgmImage;

/*画像を読み込み*/
int LoadPgmImage(char *filename, PgmImage *image, int label = -1)
{
    FILE *fp;
    char buf[128];
    int y;
    fp=fopen(filename, "rb" );
    if(fp ==NULL){
        printf("The file \"%s\" was not opened.", filename);
        return 0;
    }
    fgets(buf,128,fp);
    fgets(buf,128,fp);
    fgets(buf,128,fp);
    
    for(y=0; y<image->height; y++){
        fread(image->pixel[y], sizeof(unsigned char), image->width, fp);
    }
    image->label = label;
    fclose(fp);
    return 1;
}

/*画像保存*/
int SavePgmImage(char *filename, PgmImage *image)
{
    FILE *fp;
    int y;
    fp=fopen( filename, "wb" );
    if(fp == NULL){
        printf("The file \"%s\" was not opened.", filename);
        return 0;
    }
    fprintf(fp,"P5\n");
    fprintf(fp,"%d %d\n", image->width, image->height);
    fprintf(fp,"255\n");
    
    for(y=0; y<image->height; y++){
        fwrite(image->pixel[y], sizeof(unsigned char), image->width, fp);
    }
    fclose(fp);
    return 1;
}

/*距離計算の計算*/
/*MAX距離*/
long CalcL1Distance_m(PgmImage *img1, PgmImage *img2)
{
    long dis = 0;
    long dis_m=0;
    int x, y;
    
    for  (y=0; y<img1->height; y++){
        for (x=0; x<img1->width; x++){
            dis = abs(img1->pixel[y][x] - img2->pixel[y][x]);
            if (dis_m<dis) {
                dis_m=dis;
            }
        }
    }
    return dis_m;
}

/*距离计算*/
/*マンッハッドン距離*/
long CalcL1Distance(PgmImage *img1, PgmImage *img2)
{
    long dis = 0;
    int x, y;
    
    for  (y=0; y<img1->height; y++){
        for (x=0; x<img1->width; x++){/
            int th_val=50;
            if (img1->pixel[y][x] < th_val)
                img1->pixel[y][x] = 0;
            else
                img1->pixel[y][x] = 255;
                dis += abs(img1->pixel[y][x] - img2->pixel[y][x])*abs(img1->pixel[y][x] - img2->pixel[y][x]);
        }
    }
    return dis;
}

/*画像処理*/
/*scr原图*/
/*dst处理图*/
int Binarization(PgmImage **src, PgmImage **dst)
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

/*学習サンプル画像の読み込み*/
int LoadTemplateImages(PgmImage **tmp,PgmImage **dst)
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

/*最も近い学習用画像のクラスラベルを認識結果とする*/
int ReturnMatchLabel(PgmImage *img, PgmImage **tmp)
{
    int* k_label = new int[K_MAX];
    long* k_dist = new long[K_MAX];
    
    for (int k=0; k<K_MAX; k++) {
        k_label[k] = -1;
        k_dist[k] = INT_MAX;
    }
    
    for (int i=0; i<TrainingSampleNum; i++){
        long dis = CalcL1Distance_m(img, tmp[i]);
        
        for (int k=0; k<K_MAX; k++) {
            if (k_dist[k] > dis) {
                for (int n=K_MAX-1; n>k; n--) {
                    k_label[n] = k_label[n-1];
                    k_dist[n] = k_dist[n-1];
                }
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
    return match_label;
}

/*テスト用画像を1枚ずつ読み込み*/
/*認識結果に基づき混同行列と認識率を算出*/
int TemplateMatching(PgmImage **tmp)　//認識
{
    char filename[256];
    PgmImage *img = new PgmImage(ImageSize, ImageSize);
    RecognitionResult *result = new RecognitionResult(ClassNum);
    int each_class_num = TestSampleNum/ClassNum;
    int match_label = -1;     //認識の結果数字
    
    for (int label=0; label<ClassNum; label++){
        for (int sample=0; sample<each_class_num; sample++){
            sprintf(filename, TestDataFile, label, sample);
            printf("\rsearching the file %s\n", filename);
            if (!LoadPgmImage(filename, img, label)){
                return 0;
            }
            match_label = ReturnMatchLabel(img, tmp);	//imgテスト図，tmp訓練図
            result->res[img->label][match_label] ++;	//結果データ
        }
    }
    result->CalcRatio();
    result->PrintRecognitionResult();
    sprintf(filename, RecognitionResultFile, TrainingSampleNum, TestSampleNum,K_MAX);
    result->SaveRecognitionResult(filename,K_MAX);
    cout<<"mark"<<endl;
    delete result;
    delete img;
    return 1;
}
