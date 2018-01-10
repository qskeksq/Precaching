# PreCaching

## Bitmap PreCaching

### (1) 비동기로 비트맵 로드 클래스

```java
public class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {

    private final WeakReference imageViewReference;
    public int data = 0;

    public BitmapWorkerTask(ImageView imageView) {
        // Use a WeakReference to ensure the ImageView can be garbage collected
        imageViewReference = new WeakReference(imageView);
    }

    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        data = params[0];
//        return decodeSampledBitmapFromResource(getResources(), data, 100, 100));
        return null;
    }

    // Once complete, see if ImageView is still around and set bitmap.
    @Override
    protected void onPostExecute(Bitmap bitmap) {

        if (isCancelled()) {
            bitmap = null;
        }

        if (imageViewReference != null && bitmap != null) {
            ImageView imageView = (ImageView) imageViewReference.get();
            if (imageView != null) {
                imageView.setImageBitmap(bitmap);
            }
        }
    }

    public void loadBitmap(int resId, ImageView imageView) {
        BitmapWorkerTask task = new BitmapWorkerTask(imageView);
        task.execute(resId);
    }
}
```

```java
public class AsyncDrawable extends BitmapDrawable {
    private final WeakReference bitmapWorkerTaskReference;

    public AsyncDrawable(Resources res, Bitmap bitmap, BitmapWorkerTask bitmapWorkerTask) {
        super(res, bitmap);
        bitmapWorkerTaskReference = new WeakReference(bitmapWorkerTask);
    }

    public BitmapWorkerTask getBitmapWorkerTask() {
        return (BitmapWorkerTask) bitmapWorkerTaskReference.get();
    }

    public void loadBitmap(int resId, ImageView imageView, Context context, Bitmap mPlaceHolderBitmap) {
        if (cancelPotentialWork(resId, imageView)) {
            final BitmapWorkerTask task = new BitmapWorkerTask(imageView);
            final AsyncDrawable asyncDrawable = new AsyncDrawable(context.getResources(), mPlaceHolderBitmap, task);
            imageView.setImageDrawable(asyncDrawable);
            task.execute(resId);
        }
    }

    public boolean cancelPotentialWork(int data, ImageView imageView) {
        final BitmapWorkerTask bitmapWorkerTask = getBitmapWorkerTask(imageView);

        if (bitmapWorkerTask != null) {
            final int bitmapData = bitmapWorkerTask.data;
            if (bitmapData != data) {
                // Cancel previous task
                bitmapWorkerTask.cancel(true);
            } else {
                // The same work is already in progress
                return false;
            }
        }
        // No task associated with the ImageView, or an existing task was cancelled
        return true;
    }

    private static BitmapWorkerTask getBitmapWorkerTask(ImageView imageView) {
        if (imageView != null) {
            final Drawable drawable = imageView.getDrawable();
            if (drawable instanceof AsyncDrawable) {
                final AsyncDrawable asyncDrawable = (AsyncDrawable) drawable;
                return asyncDrawable.getBitmapWorkerTask();
            }
        }
        return null;
    }
}
```

### (2) LruCache 리스트 생성

```java
LruCache<String, Bitmap> memCache;
final int memClass = ((ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE)).getMemoryClass();
final int cacheSize = 1024 * 1024 * memClass / 8;
memCache = new LruCache<String, Bitmap>(cacheSize) {
    @Override
    protected int sizeOf(String key, Bitmap bitmap) {
        return bitmap.getByteCount();
    }
};
```

### (3) 글라이드 비트맵 Precaching

```java
// 글라이드 자체적으로도 preCaching 해줌
Glide.with(context)
        .load(imageFiles.get(i))
        .diskCacheStrategy(DiskCacheStrategy.SOURCE)
        .preload();
final String filename = couponList.get(i).logoFilename;
// 비트맵이 다 로드 된 후 비동기로 메모리에 캐싱
Glide.with(context)
        .load(imageFiles.get(i))
        .asBitmap()
        .fitCenter() //fits given dimensions maintaining ratio
        .into(new SimpleTarget<Bitmap>() {
            @Override
            public void onResourceReady(Bitmap resource, GlideAnimation<? super Bitmap> glideAnimation) {
                memCache.put(filename, resource);
            }
        });
```

### (4) 비트맵으로 Preload 호출

```java
public void addBitmapToMemoryCache(String key, Bitmap bitmap) {
    if (getBitmapFromMemCache(key) == null) {
        memCache.put(key, bitmap);
    }
}

public Bitmap getBitmapFromMemCache(String key) {
    return memCache.get(key);
}

public void loadBitmap(int resId, ImageView mImageView) {
    final String imageKey = String.valueOf(resId);

    final Bitmap bitmap = getBitmapFromMemCache(imageKey);
    if (bitmap != null) {
        mImageView.setImageBitmap(bitmap);
    } else {
        BitmapWorkerTask task = new BitmapWorkerTask(mImageView);
        task.execute(resId);
    }
}

```

### (5) 적용

```java
Bitmap image = memCache.get(couponList.get(position).logoFilename);
if (image != null) {
    holder.couponLogo.setImageBitmap(image);
    Log.e("호출", "캐시사용");
} else {
    Glide.with(context).load(imageFiles.get(position)).into(holder.couponLogo);
    Log.e("호출", "캐시미사용");
}
loadBitmap();
```

## RecyclerView Precaching

### (1) PreCachingLayoutManager

```java
public class PreCachingLayoutManager extends LinearLayoutManager {
    private static final int DEFAULT_EXTRA_LAYOUT_SPACE = 600;
    private int extraLayoutSpace = -1;

    public PreCachingLayoutManager(Context context) {
        super(context);
    }

    public void setExtraLayoutSpace(int extraLayoutSpace) {
        this.extraLayoutSpace = extraLayoutSpace;
    }

    @Override
    protected int getExtraLayoutSpace(RecyclerView.State state) {
        if (extraLayoutSpace > 0) {
            return extraLayoutSpace;
        }
        return DEFAULT_EXTRA_LAYOUT_SPACE;
    }
}
```

### (2) 적용

```java
PreCachingLayoutManager manager = new PreCachingLayoutManager(this);
manager.setItemPrefetchEnabled(true);
manager.setInitialPrefetchItemCount(10);
manager.setOrientation(LinearLayoutManager.VERTICAL);
manager.setExtraLayoutSpace(2000);
```
