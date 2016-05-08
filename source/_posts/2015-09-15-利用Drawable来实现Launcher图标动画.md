title: 利用Drawable来实现Launcher图标动画

date: 2015-09-15 21:05:32
updated: 2015-09-20 20:22:20
comments: true
categories:
- Diary
tags:
- Android
-------------

目标：实现桌面的时钟图标的更换，以及实现实时显示时间
----------------------------------------------------

### 方案

替换Launcher中特定应用的图标有三种实现方法：

-	[在framework中修改] 这种修改方式可以实现全局的修改，也就是修改一处，实现整个系统也可以修改。
-	[利用view的onDraw进行修改] 这种方式是采用在Launcher启动的时候进行广播的注册，并且在在回调接口中 添加回调函数，并让系统进行调用，利用view来进行更新，这种方式，也就是不会影响系统的一些图标的效果， 但是通过view来修改图标，这样就增加的系统的消耗。
-	[利用Drawable实现Launcher图标的更新] 这种方式是让与view关联的drawable自身进行更新，这样就可以 跳过view，可以减少系统的消耗，但是要考虑的是否影响了一些图标的效果。

### 编写背景

-	1、Android 5.1
-	2、Launcher3

### 思路

-	1、首先要实现时钟的动态跟新，我们需要监听系统广播，这里有三种广播，分别为：
	-	a）ACTION_TIME_TICK
	-	b）ACTION_TIME_CHANGED，
	-	c）ACTION_TIMEZONE_CHANGED
-	2、对这个三个广播做出响应，并获得时间，并进行重绘（这里利用invalidateSelf()方法进行自我重绘）
-	3、重写Drawable的四个方法:
	-	a) draw()
	-	b) getOpacity()
	-	c) setAlpha()
	-	d) setColorFilter()  
		这里draw()是才调用invalidateSelf()后，系统会进行调用的方法，而且这也是主要进行绘画动作 的地方。  
		然后就是其他两个setter方法，这也是相对重要的，因为在Launcher3中是使用ColorFilter来更新 画笔的颜色来实现图标的明暗变化，所以在setColorFilter()的实现的时候需要考虑图标是否被点击。
-	4、对系统绘画图标时进行替换,

### 实现概要

-	1、编写一个中间抽象类AnimationDrawable来继承FastBitmapDrawable()，主要是Launcher3中图标（PS：BubbleTextView）的Drawable主要是采用FastBitmapDrawable，这也是为了尽可能少的修改Launcher3的代码。
-	2、在AnimationDrawable中定义两个抽象的方法，这是为了广播的管理（PS：也就是注册和取消注册）：
	-	a) public abstract void registerBroadCast();
	-	b) public abstract void unregisterBroadCast();
-	3、时间的更新，并且进行invalidateSelf()来自我更新。

### 具体实现

-	1、广播接受者：

	```
	private final BroadcastReceiver clockReceiver = new BroadcastReceiver() {
	    @Override
	    public void onReceive(Context context, Intent intent) {
	        if (intent.getAction().equals(Intent.ACTION_TIMEZONE_CHANGED)) {
	            String tz = intent.getStringExtra("time-zone");
	            mTimeZoneId = TimeZone.getTimeZone(tz).getID();
	        }
	        onTimeChanged();
	    }
	};
	```

-	2、时间相关方法

	```
	private void onTimeChanged() {
	    Calendar calendar = Calendar.getInstance();
	    calendar.setTimeInMillis(System.currentTimeMillis());
	    if (mTimeZoneId != null) {
	        calendar.setTimeZone(TimeZone.getTimeZone(mTimeZoneId));
	    }

	    int hour = calendar.get(Calendar.HOUR);
	    int minute = calendar.get(Calendar.MINUTE);
	    int second = calendar.get(Calendar.SECOND);

	    mMinutes = minute + second / 60.0f;
	    mHour = hour + mMinutes / 60.0f;
	    mChanged = true;
	    invalidateSelf();
	}
	```

-	3、draw()方法的重写

	```
	@Override
	public void draw(Canvas canvas) {
	    final Rect bounds = getBounds();

	    // draw backgroud picture;
	    canvas.save();
	    canvas.drawBitmap(mBitmap, null, bounds, mPaint);
	    canvas.restore();

	    int availableWidth = bounds.right - bounds.left;
	    int availableHeight = bounds.bottom - bounds.top;
	    if (mIntrinsicWidth > 0 && mIntrinsicHeight > 0) {
	        availableWidth = mIntrinsicWidth;
	        availableHeight = mIntrinsicHeight;
	    }
	    int x = availableWidth / 2;
	    int y = availableHeight / 2;
	    float radius = Math.min(x, y) * RADIUS_SCALE;

	    //draw hour;
	    canvas.save();
	    //这里采用新的Paint是为了不破坏原本在FastBitmapDrawable中的Paint实例
	    Paint timePaint = new Paint(mPaint);
	    timePaint.setColor(Color.WHITE);
	    timePaint.setStyle(Paint.Style.STROKE);
	    timePaint.setAntiAlias(true);
	    float hourRotation = mHour / 12.0f * 360;
	    canvas.rotate(hourRotation, x, y);
	    canvas.drawLine(x, y + (radius * OFFSET_RADIUS_SCALE),
	            x, y - (radius * HOUR_RADIUS_SCALE), timePaint);
	    canvas.restore();
	    //draw min
	    canvas.save();
	    float minRotation = mMinutes / 60f * 360;
	    canvas.rotate(minRotation, x, y);
	    canvas.drawLine(x, y + (radius * OFFSET_RADIUS_SCALE),
	            x, y - (radius * MINUTE_RADIUS_SCALE), timePaint);
	    canvas.restore();
	}
	```

### 缺陷

当有点击效果的时候，用于draw方法十分耗时，但是点击效果不明显。
