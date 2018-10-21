---
title: Android OCR之Tess-two
date: 2017-11-25 14:48:16
tags:
---
实现小猿搜题、作业帮类似效果。
基于Google Tesseract-OCR实现，由于这是基于C++开发，Android中不能直接使用，所以本项目使用tess-two是对于Android的分支。

## 准备工作
### 1）Android Studio导入gradle依赖（快速集成）
```
//编译好的SO库 和 jar包
compile 'com.rmtheis:tess-two:6.1.1'
//图像裁剪
compile 'com.edmodo:cropper:1.0.1' 
```
>其实也可以自己下载[https://github.com/rmtheis/tess-two](https://github.com/rmtheis/tess-two) 源码在Linux环境中进行编译。

### 2）第二种方法 自己编译源码（！！！）
- 安装虚拟机VMware
- 安装linux系统Ubunt
- 安装必要工具

```
sudo apt-get update
sudo apt-get install git
```
- 配置JDK、NDK、SDK环境（踩了一万个坑）[具体教程地址](https://wangtaot.github.io/2017/11/25/Android%E7%BC%96%E8%AF%91so%E5%BA%93/)
- 下载tess-two代码
- 开始编译(依次输入以下两行命令)

```
cd /tess/tess-two/jni
ndk-build
```
- 使用
将编译好的tess-two目录复制到自己项目的libraries下，将项目下的app关联 libraries。

--------------------
## tess-two使用

- 数据字典下载，在app/main目录下新建assets目录，从https://github.com/tesseract-ocr/tessdata  下载对应语言的字典库。

- MainActivity.java

>主要就是将assets中的字典文件写到手机文件目录

```
/**
 * 将assets中的文件复制出
 *
 * @param path
 */
public void deepFile(String path) {
    String newPath = getExternalFilesDir(null) + "/";
    try {
        String str[] = getAssets().list(path);
        if (str.length > 0) {//如果是目录
            File file = new File(newPath + path);
            file.mkdirs();
            for (String string : str) {
                path = path + "/" + string;
                deepFile(path);
                path = path.substring(0, path.lastIndexOf('/'));//回到原来的path
            }
        } else {//如果是文件
            InputStream is = getAssets().open(path);
            FileOutputStream fos = new FileOutputStream(new File(newPath + path));
            byte[] buffer = new byte[1024];
            int count = 0;
            while (true) {
                count++;
                int len = is.read(buffer);
                if (len == -1) {
                    break;
                }
                fos.write(buffer, 0, len);
            }
            is.close();
            fos.close();
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```
>点击跳转到拍照界面

```
public void onClick(View view) {
        switch (view.getId()) {
            case R.id.btn_camera:
                Intent intent = new Intent(this, TakePhoteActivity.class);
                startActivity(intent);
                break;
        }
    }
```

- TakePhoteActivity.java 主要执行拍照、裁剪操作。

>以下是拍照成功后回掉接口，拍照成功后显示剪裁界面

```
/**
    * 拍照成功后回调
    * 存储图片并显示截图界面
    *
    * @param data
    */
   @Override
   public void onCameraStopped(byte[] data) {
       Log.i("TAG", "==onCameraStopped==");
       // 创建图像
       Bitmap bitmap = BitmapFactory.decodeByteArray(data, 0, data.length);
       // 系统时间
       long dateTaken = System.currentTimeMillis();
       // 图像名称
       String filename = DateFormat.format("yyyy-MM-dd kk.mm.ss", dateTaken).toString() + ".jpg";
       // 存储图像（PATH目录）
       Uri source = insertImage(getContentResolver(), filename, dateTaken, PATH, filename, bitmap, data);
       //准备截图
       try {
           mCropImageView.setImageBitmap(MediaStore.Images.Media.getBitmap(this.getContentResolver(), source));
       } catch (IOException e) {
           Log.e(TAG, e.getMessage());
       }
       showCropperLayout();
   }
```
>剪裁操作，完成后跳转到识别界面

```
private View.OnClickListener cropcper = new View.OnClickListener() {
        @Override
        public void onClick(View view) {
            switch (view.getId()) {
                case R.id.btn_closecropper:
                    showTakePhotoLayout();
                    break;
                case R.id.btn_startcropper:
                    //获取截图并旋转90度
                    Bitmap cropperBitmap = mCropImageView.getCroppedImage();
                    Bitmap bitmap = Utils.rotate(cropperBitmap, -90);
                    // 系统时间
                    long dateTaken = System.currentTimeMillis();
                    // 图像名称
                    String filename = DateFormat.format("yyyy-MM-dd kk.mm.ss", dateTaken).toString() + ".jpg";
                    Uri uri = insertImage(getContentResolver(), filename, dateTaken, PATH, filename, bitmap, null);
                    Intent intent = new Intent(TakePhoteActivity.this, ShowCropperedActivity.class);
                    intent.setData(uri);
                    intent.putExtra("path", PATH + filename);
                    intent.putExtra("width", bitmap.getWidth());
                    intent.putExtra("height", bitmap.getHeight());
//                  intent.putExtra("cropperImage", bitmap);
                    startActivity(intent);
                    bitmap.recycle();
                    finish();
                    break;
            }
        }
    };
```

- ShowCropperedActivity.java 主要是识别操作

>初始化识别器

```
//sd卡路径
private static String LANGUAGE_PATH = "";
//识别语言
private static final String LANGUAGE = "chi_sim";//chi_sim | eng
private TessBaseAPI baseApi = new TessBaseAPI();
@Override
protected void onCreate(Bundle savedInstanceState) {
    LANGUAGE_PATH = getExternalFilesDir("") + "/";
    
    baseApi.init(LANGUAGE_PATH, LANGUAGE);
    //设置设别模式
    baseApi.setPageSegMode(TessBaseAPI.PageSegMode.PSM_AUTO);
}
```
>将图片灰度化处理，去除杂色可以提高准确度

```
/**
 * 灰度化处理
 *
 * @param bitmap3
 * @return
 */
public Bitmap convertGray(Bitmap bitmap3) {
    colorMatrix = new ColorMatrix();
    colorMatrix.setSaturation(0);
    ColorMatrixColorFilter filter = new ColorMatrixColorFilter(colorMatrix);
    Paint paint = new Paint();
    paint.setColorFilter(filter);
    Bitmap result = Bitmap.createBitmap(bitmap3.getWidth(), bitmap3.getHeight(), Bitmap.Config.ARGB_8888);
    Canvas canvas = new Canvas(result);
    canvas.drawBitmap(bitmap3, 0, 0, paint);
    return result;
}
```
>开始识别

```
//传入图片
baseApi.setImage(bitmap);
//获取识别后的结果
String result = baseApi.getUTF8Text();
//结束识别
baseApi.end();
```
--------------

## 结果
Demo已上传Github如需要可下载https://github.com/wangtaoT/AndroidOCR
![](Android-OCR之Tess-two/result.jpg) 