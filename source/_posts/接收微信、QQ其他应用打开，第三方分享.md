---
title: Android接收微信、QQ其他应用打开，第三方分享
date: 2020-03-27 10:40:41
tags:
---

### 在AndroidManifest.xml注册ACTION事件


```
<activity
        android:name="com.test.app.MainActivity"
        android:configChanges="orientation|keyboardHidden|screenSize"
        android:label="这里的名称会对外显示"
        android:launchMode="singleTask"
        android:screenOrientation="portrait">
        //注册接收分享
        <intent-filter>
            <action android:name="android.intent.action.SEND" />
            <category android:name="android.intent.category.DEFAULT" />

            //接收分享的文件类型
            <data android:mimeType="image/*" />
            <data android:mimeType="application/msword" />
            <data android:mimeType="application/vnd.openxmlformats-officedocument.wordprocessingml.document" />
            <data android:mimeType="application/vnd.ms-excel" />
            <data android:mimeType="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" />
            <data android:mimeType="application/vnd.ms-powerpoint" />
            <data android:mimeType="application/vnd.openxmlformats-officedocument.presentationml.presentation" />
            <data android:mimeType="application/pdf" />
            <data android:mimeType="text/plain" />
            </intent-filter>
            
        //注册默认打开事件，微信、QQ的其他应用打开
        <intent-filter>
            <action android:name="android.intent.action.VIEW" />
            <category android:name="android.intent.category.DEFAULT" />
            
            //接收打开的文件类型
            <data android:scheme="file" />
            <data android:scheme="content" />
            <data android:mimeType="image/*" />
            <data android:mimeType="application/msword" />
            <data android:mimeType="application/vnd.openxmlformats-officedocument.wordprocessingml.document" />
            <data android:mimeType="application/vnd.ms-excel" />
            <data android:mimeType="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" />
            <data android:mimeType="application/vnd.ms-powerpoint" />
            <data android:mimeType="application/vnd.openxmlformats-officedocument.presentationml.presentation" />
            <data android:mimeType="application/pdf" />
            <data android:mimeType="text/plain" />
            </intent-filter>
        </activity>
```

### 在用于接收分享的Activity里面加接收代码

1. 当APP进程在后台时，会调用Activity的onNewIntent方法
2. 当APP进程被杀死时，会调用onCreate方法

> 所以在两个方法中都需要监听事件

```
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        receiveActionSend(intent);
    }
    
    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        receiveActionSend(intent);
    }
```
> receiveActionSend方法如下

```
    public void receiveActionSend(Intent intent) {
        String action = intent.getAction();
        String type = intent.getType();

        //判断action事件
        if (type == null || (!Intent.ACTION_VIEW.equals(action) && !Intent.ACTION_SEND.equals(action))) {
            return;
        }
        
        //取出文件uri
        Uri uri = intent.getData();
        if (uri == null) {
            uri = intent.getParcelableExtra(Intent.EXTRA_STREAM);
        }
        
        //获取文件真实地址
        String filePath = UriUtils.getFileFromUri(EdusohoApp.baseApp, uri);
        if (TextUtils.isEmpty(filePath)) {
            return;
        }

        //业务处理
        .
        .
        .
    }
```

> 获取真实路径getFileFromUri方法

```
    /**
     * 获取真实路径
     *
     * @param context
     */
    public static String getFileFromUri(Context context, Uri uri) {
        if (uri == null) {
            return null;
        }
        switch (uri.getScheme()) {
            case ContentResolver.SCHEME_CONTENT:
                //Android7.0之后的uri content:// URI
                return getFilePathFromContentUri(context, uri);
            case ContentResolver.SCHEME_FILE:
            default:
                //Android7.0之前的uri file://
                return new File(uri.getPath()).getAbsolutePath();
        }
    }
```
> Android7.0之后的uri content:// URI需要对微信、QQ等第三方APP做兼容

- 在文件管理选择本应用打开时,url的值为content://media/external/file/85139  
- 在微信中选择本应用打开时,url的值为
content://com.tencent.mm.external.fileprovider/external/tencent/MicroMsg/Download/111.doc  
- 在QQ中选择本应用打开时,url的值为
content://com.tencent.mobileqq.fileprovider/external_files/storage/emulated/0/Tencent/QQfile_recv/

> 第一种为系统统一文件资源，能通过系统方法转化为绝对路径；    
微信、QQ的为fileProvider，只能获取到文件流，需要先将文件copy到自己的私有目录。   
方法如下：

```
    /**
     * 从uri获取path
     *
     * @param uri content://media/external/file/109009
     *            <p>
     *            FileProvider适配
     *            content://com.tencent.mobileqq.fileprovider/external_files/storage/emulated/0/Tencent/QQfile_recv/
     *            content://com.tencent.mm.external.fileprovider/external/tencent/MicroMsg/Download/
     */
    private static String getFilePathFromContentUri(Context context, Uri uri) {
        if (null == uri) return null;
        String data = null;

        String[] filePathColumn = {MediaStore.MediaColumns.DATA, MediaStore.MediaColumns.DISPLAY_NAME};
        Cursor cursor = context.getContentResolver().query(uri, filePathColumn, null, null, null);
        if (null != cursor) {
            if (cursor.moveToFirst()) {
                int index = cursor.getColumnIndex(MediaStore.MediaColumns.DATA);
                if (index > -1) {
                    data = cursor.getString(index);
                } else {
                    int nameIndex = cursor.getColumnIndex(MediaStore.MediaColumns.DISPLAY_NAME);
                    String fileName = cursor.getString(nameIndex);
                    data = getPathFromInputStreamUri(context, uri, fileName);
                }
            }
            cursor.close();
        }
        return data;
    }
    
    /**
     * 用流拷贝文件一份到自己APP私有目录下
     *
     * @param context
     * @param uri
     * @param fileName
     */
    private static String getPathFromInputStreamUri(Context context, Uri uri, String fileName) {
        InputStream inputStream = null;
        String filePath = null;

        if (uri.getAuthority() != null) {
            try {
                inputStream = context.getContentResolver().openInputStream(uri);
                File file = createTemporalFileFrom(context, inputStream, fileName);
                filePath = file.getPath();

            } catch (Exception e) {
            } finally {
                try {
                    if (inputStream != null) {
                        inputStream.close();
                    }
                } catch (Exception e) {
                }
            }
        }

        return filePath;
    }

    private static File createTemporalFileFrom(Context context, InputStream inputStream, String fileName)
            throws IOException {
        File targetFile = null;

        if (inputStream != null) {
            int read;
            byte[] buffer = new byte[8 * 1024];
            //自己定义拷贝文件路径
            targetFile = new File(context.getExternalCacheDir(), fileName);
            if (targetFile.exists()) {
                targetFile.delete();
            }
            OutputStream outputStream = new FileOutputStream(targetFile);

            while ((read = inputStream.read(buffer)) != -1) {
                outputStream.write(buffer, 0, read);
            }
            outputStream.flush();

            try {
                outputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        return targetFile;
    }
```
