# RQrCodeDemo
个人认为开源二维码扫描封装库中, 最快的.    请随手点个Star. 谢谢!

**推荐使用 https://github.com/XieZhiFa/ZxingZbar**
- zbar 快, 支持的格式少.
- zxing 慢, 支持的格式多.

替换`DecodeHandler`类为已下内容 就可以实现`zbar`和`zxing`交替识别二维码.
```java
final class DecodeHandler extends Handler {

    private static final String TAG = DecodeHandler.class.getSimpleName();

    private final IActivity activity;
    private final MultiFormatReader multiFormatReader;
    DecoderMode mDecoderMode = DecoderMode.Zbar; //默认扫码类型, 自动切换
    long lastDecoderTime = 0L;
    private boolean running = true;

    DecodeHandler(IActivity activity, Map<DecodeHintType, Object> hints) {
        multiFormatReader = new MultiFormatReader();
        multiFormatReader.setHints(hints);
        this.activity = activity;
    }

    private static void bundleThumbnail(PlanarYUVLuminanceSource source, Bundle bundle) {
        int[] pixels = source.renderThumbnail();
        int width = source.getThumbnailWidth();
        int height = source.getThumbnailHeight();
        Bitmap bitmap = Bitmap.createBitmap(pixels, 0, width, width, height, Bitmap.Config.ARGB_8888);
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        bitmap.compress(Bitmap.CompressFormat.JPEG, 50, out);
        bundle.putByteArray(DecodeThread.BARCODE_BITMAP, out.toByteArray());
        bundle.putFloat(DecodeThread.BARCODE_SCALED_FACTOR, (float) width / source.getWidth());
    }

    @Override
    public void handleMessage(Message message) {
        if (message == null || !running) {
            return;
        }
        if (message.what == R.id.decode) {
            decode((byte[]) message.obj, message.arg1, message.arg2);

        } else if (message.what == R.id.quit) {
            running = false;
            Looper.myLooper().quit();
        }
    }

    private void changeDecodeMode() {
        if (mDecoderMode == DecoderMode.Zxing) {
            mDecoderMode = DecoderMode.Zbar;
        } else {
            mDecoderMode = DecoderMode.Zxing;
        }
    }

    /**
     * Decode the data within the viewfinder rectangle, and time how long it took. For efficiency,
     * reuse the same reader objects from one decode to the next.
     *
     * @param data   The YUV preview frame.
     * @param width  The width of the preview frame.
     * @param height The height of the preview frame.
     */
    private void decode(byte[] data, int width, int height) {
        long start = System.currentTimeMillis();

        if (lastDecoderTime == 0) {
            lastDecoderTime = start;
        }

        String resultQRcode = null;

        if (mDecoderMode == DecoderMode.Zxing) { //如果扫描模式是Zxing
//            // --- add java进行数组的转换 速度很慢
//            byte[] rotatedData = new byte[data.length];
//            for (int y = 0; y < height; y++) {
//                for (int x = 0; x < width; x++)
//                    rotatedData[x * height + height - y - 1] = data[x + y * width];
//            }
//            int tmp = width;
//            width = height;
//            height = tmp;
//            data = rotatedData;
//            Log.d(TAG, "数组转换用时: " + (System.currentTimeMillis() - start));
//            //--- end


            /*
              因为相机传感器捕获的数据是横向的, 所以需要将数据进行90度的旋转, 用java进行转换在红米三手机测试大概需要 600ms左右
              因此换了C语言, 只需要 35ms左右 速度快了接近 20倍
             */
            data = DecodeHandlerJni.dataHandler(data, data.length, width, height);
            //Log.d(TAG, "数组转换用时: " + (System.currentTimeMillis() - start));
            int tmp = width;
            width = height;
            height = tmp;

            Result rawResult = null;
            PlanarYUVLuminanceSource source = activity.getCameraManager().buildLuminanceSource(data, width, height);
            if (source != null) {
                BinaryBitmap bitmap = new BinaryBitmap(new HybridBinarizer(source));
                try {
                    rawResult = multiFormatReader.decodeWithState(bitmap);
                } catch (ReaderException re) {
                    // continue
                } finally {
                    multiFormatReader.reset();
                }
            }
            if (rawResult != null) {
                resultQRcode = rawResult.getText();
            }

        } else {
            Image barcode = new Image(width, height, "Y800");
            barcode.setData(data);
            Rect rect = activity.getCameraManager().getFramingRectInPreview();
            if (rect != null) {
                /*
                    zbar 解码库,不需要将数据进行旋转,因此设置裁剪区域是的x为 top, y为left
                    设置了裁剪区域,解码速度快了近5倍左右
                 */
                barcode.setCrop(rect.top, rect.left, rect.width(), rect.height());    // 设置截取区域，也就是你的扫描框在图片上的区域.
            }
            ImageScanner mImageScanner = new ImageScanner();
            int result = mImageScanner.scanImage(barcode);
            if (result != 0) {
                SymbolSet symSet = mImageScanner.getResults();
                for (Symbol sym : symSet)
                    resultQRcode = sym.getData();
            }
        }

        long end = System.currentTimeMillis();
        if (BuildConfig.DEBUG) {
            Log.d(TAG, "处理帧 " + (end - start) + " ms " + mDecoderMode);
        }

        if ((end - start) < 1_000 && end - lastDecoderTime > 1_000) {
            //2秒之后切换模式
            changeDecodeMode();
            lastDecoderTime = end;
        }

        Handler handler = activity.getHandler();
        if (!TextUtils.isEmpty(resultQRcode)) {   // 非空表示识别出结果了。
            if (handler != null) {
                Log.d(TAG, "解码成功: " + resultQRcode);
                Message message = Message.obtain(handler, R.id.decode_succeeded, resultQRcode);
                message.sendToTarget();
            }
        } else {
            if (handler != null) {
                Message message = Message.obtain(handler, R.id.decode_failed);
                message.sendToTarget();
            }
        }
    }
}

```


# 联系作者
> 请使用QQ扫码加群, 小伙伴们在等着你哦!

![](https://raw.githubusercontent.com/angcyo/res/master/image/qq/qq_group_code.png)

> 关注我的公众号, 每天都能一起玩耍哦!

![](https://raw.githubusercontent.com/angcyo/res/master/image/weixin/%E8%AE%A2%E9%98%85%E5%8F%B7_%E4%BA%8C%E7%BB%B4%E7%A0%81/qrcode_for_gh_59fa6d9a51d8_258_8cm.jpg)
