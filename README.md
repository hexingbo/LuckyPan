# LuckyPan
慕课网上的一个抽奖转盘的实现源码

# 效果图
![LuckyPan](http://7xq7wz.com1.z0.glb.clouddn.com/luckyPan.gif)
3、效果图


就这么个效果，当然了模拟器录制的效果肯定没有真机上效果流畅。

结合上面我们给出的模版，我们需要改变的就是，在create回调里面需要去初始化一些变量，在draw方法里面去绘制我们的文本、图片、扇形块块等等。整体架构没有变化。

4、转盘的制作

1、构造方法以及变量
public class LuckyPanView extends SurfaceView implements Callback, Runnable
{
 
	private SurfaceHolder mHolder;
	/**
	 * 与SurfaceHolder绑定的Canvas
	 */
	private Canvas mCanvas;
	/**
	 * 用于绘制的线程
	 */
	private Thread t;
	/**
	 * 线程的控制开关
	 */
	private boolean isRunning;
 
	/**
	 * 抽奖的文字
	 */
	private String[] mStrs = new String[] { "单反相机", "IPAD", "恭喜发财", "IPHONE",
			"妹子一只", "恭喜发财" };
	/**
	 * 每个盘块的颜色
	 */
	private int[] mColors = new int[] { 0xFFFFC300, 0xFFF17E01, 0xFFFFC300,
			0xFFF17E01, 0xFFFFC300, 0xFFF17E01 };
	/**
	 * 与文字对应的图片
	 */
	private int[] mImgs = new int[] { R.drawable.danfan, R.drawable.ipad,
			R.drawable.f040, R.drawable.iphone, R.drawable.meizi,
			R.drawable.f040 };
 
	/**
	 * 与文字对应图片的bitmap数组
	 */
	private Bitmap[] mImgsBitmap;
	/**
	 * 盘块的个数
	 */
	private int mItemCount = 6;
 
	/**
	 * 绘制盘块的范围
	 */
	private RectF mRange = new RectF();
	/**
	 * 圆的直径
	 */
	private int mRadius;
	/**
	 * 绘制盘快的画笔
	 */
	private Paint mArcPaint;
 
	/**
	 * 绘制文字的画笔
	 */
	private Paint mTextPaint;
 
	/**
	 * 滚动的速度
	 */
	private double mSpeed;
	private volatile float mStartAngle = 0;
	/**
	 * 是否点击了停止
	 */
	private boolean isShouldEnd;
 
	/**
	 * 控件的中心位置
	 */
	private int mCenter;
	/**
	 * 控件的padding，这里我们认为4个padding的值一致，以paddingleft为标准
	 */
	private int mPadding;
 
	/**
	 * 背景图的bitmap
	 */
	private Bitmap mBgBitmap = BitmapFactory.decodeResource(getResources(),
			R.drawable.bg2);
	/**
	 * 文字的大小
	 */
	private float mTextSize = TypedValue.applyDimension(
			TypedValue.COMPLEX_UNIT_SP, 20, getResources().getDisplayMetrics());
 
	public LuckyPanView(Context context)
	{
		this(context, null);
	}
 
	public LuckyPanView(Context context, AttributeSet attrs)
	{
		super(context, attrs);
 
		mHolder = getHolder();
		mHolder.addCallback(this);
 
		// setZOrderOnTop(true);// 设置画布 背景透明
		// mHolder.setFormat(PixelFormat.TRANSLUCENT);
 
		setFocusable(true);
		setFocusableInTouchMode(true);
		this.setKeepScreenOn(true);
 
	}

我们在构造中设置了Callback回调，然后通过成员变量，大家应该也能看得出来每个变量的作用，以及可能有的代码快。
2、onMeasure
这里我简单重写了一下onMeasure，使我们的控件为正方形

/**
	 * 设置控件为正方形
	 */
	@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec)
	{
		super.onMeasure(widthMeasureSpec, heightMeasureSpec);
 
		int width = Math.min(getMeasuredWidth(), getMeasuredHeight());
		// 获取圆形的直径
		mRadius = width - getPaddingLeft() - getPaddingRight();
		// padding值
		mPadding = getPaddingLeft();
		// 中心点
		mCenter = width / 2;
		setMeasuredDimension(width, width);
	}

并且为我们的mRadius和mCenter进行了赋值。
3、surfaceCreated
@Override
	public void surfaceCreated(SurfaceHolder holder)
	{
		// 初始化绘制圆弧的画笔
		mArcPaint = new Paint();
		mArcPaint.setAntiAlias(true);
		mArcPaint.setDither(true);
		// 初始化绘制文字的画笔
		mTextPaint = new Paint();
		mTextPaint.setColor(0xFFffffff);
		mTextPaint.setTextSize(mTextSize);
		// 圆弧的绘制范围
		mRange = new RectF(getPaddingLeft(), getPaddingLeft(), mRadius
				+ getPaddingLeft(), mRadius + getPaddingLeft());
 
		// 初始化图片
		mImgsBitmap = new Bitmap[mItemCount];
		for (int i = 0; i < mItemCount; i++)
		{
			mImgsBitmap[i] = BitmapFactory.decodeResource(getResources(),
					mImgs[i]);
		}
 
		// 开启线程
		isRunning = true;
		t = new Thread(this);
		t.start();
	}

surfaceCreated我们初始化了绘制需要用到的变量，以及开启了线程。
surfaceDestroyed中就一行代码，顺便贴出。

@Override
	public void surfaceChanged(SurfaceHolder holder, int format, int width,
			int height)
	{
		// TODO Auto-generated method stub
 
	}
 
	@Override
	public void surfaceDestroyed(SurfaceHolder holder)
	{
		// 通知关闭线程
		isRunning = false;
	}

可以猜到核心的代码都在我们的线程的run里面了。
4、draw
@Override
	public void run()
	{
		// 不断的进行draw
		while (isRunning)
		{
			long start = System.currentTimeMillis();
			draw();
			long end = System.currentTimeMillis();
			try
			{
				if (end - start < 50)
				{
					Thread.sleep(50 - (end - start));
				}
			} catch (InterruptedException e)
			{
				e.printStackTrace();
			}
 
		}
 
	}
 
	private void draw()
	{
		try
		{
			// 获得canvas
			mCanvas = mHolder.lockCanvas();
			if (mCanvas != null)
			{
				// 绘制背景图
				drawBg();
 
				/**
				 * 绘制每个块块，每个块块上的文本，每个块块上的图片
				 */
				float tmpAngle = mStartAngle;
				float sweepAngle = (float) (360 / mItemCount);
				for (int i = 0; i < mItemCount; i++)
				{
					// 绘制快快
					mArcPaint.setColor(mColors[i]);
					mCanvas.drawArc(mRange, tmpAngle, sweepAngle, true,
							mArcPaint);
					// 绘制文本
					drawText(tmpAngle, sweepAngle, mStrs[i]);
					// 绘制Icon
					drawIcon(tmpAngle, i);
 
					tmpAngle += sweepAngle;
				}
 
				// 如果mSpeed不等于0，则相当于在滚动
				mStartAngle += mSpeed;
 
				// 点击停止时，设置mSpeed为递减，为0值转盘停止
				if (isShouldEnd)
				{
					mSpeed -= 1;
				}
				if (mSpeed <= 0)
				{
					mSpeed = 0;
					isShouldEnd = false;
				}
				// 根据当前旋转的mStartAngle计算当前滚动到的区域
				calInExactArea(mStartAngle);
			}
		} catch (Exception e)
		{
			e.printStackTrace();
		} finally
		{
			if (mCanvas != null)
				mHolder.unlockCanvasAndPost(mCanvas);
		}
 
	}

可以看到我们的run里面调用了draw，和上面模版一致。
使用通过 mHolder.lockCanvas();获得我们的Canvas，然后就可以尽情的绘制了。

1、绘制背景drawBg();
	/**
	 * 根据当前旋转的mStartAngle计算当前滚动到的区域 绘制背景，不重要，完全为了美观
	 */
	private void drawBg()
	{
		mCanvas.drawColor(0xFFFFFFFF);
		mCanvas.drawBitmap(mBgBitmap, null, new Rect(mPadding / 2,
				mPadding / 2, getMeasuredWidth() - mPadding / 2,
				getMeasuredWidth() - mPadding / 2), null);
	}

这个比较简单，其实就是绘制一个棕色的圆盘，在运行代码前，你可以忽略掉，不影响。
接下来一个for循环，且角度每次递增(360 / mItemCount);就是绘制每个盘块以及盘块上的字体和图标了。

2、绘制盘块
// 绘制快快
					mArcPaint.setColor(mColors[i]);
					mCanvas.drawArc(mRange, tmpAngle, sweepAngle, true,
							mArcPaint);

这个比较简单了~~
3、绘制文本
/**
	 * 绘制文本
	 * 
	 * @param rect
	 * @param startAngle
	 * @param sweepAngle
	 * @param string
	 */
	private void drawText(float startAngle, float sweepAngle, String string)
	{
		Path path = new Path();
		path.addArc(mRange, startAngle, sweepAngle);
		float textWidth = mTextPaint.measureText(string);
		// 利用水平偏移让文字居中
		float hOffset = (float) (mRadius * Math.PI / mItemCount / 2 - textWidth / 2);// 水平偏移
		float vOffset = mRadius / 2 / 6;// 垂直偏移
		mCanvas.drawTextOnPath(string, path, hOffset, vOffset, mTextPaint);
	}

利用Path，添加入一个Arc，然后设置水平和垂直的偏移量，垂直偏移量就是当前Arc朝着圆心移动的距离；水平偏移量，就是顺时针去旋转，
我们偏移了 (mRadius * Math.PI / mItemCount / 2 - textWidth / 2);目的是为了文字居中。mRadius * Math.PI 是圆的周长；周长/ mItemCount / 2 是每个Arc的一半的长度；

拿Arc一半的长度减去textWidth / 2，就把文字设置居中了。

最后，用过path去绘制文本即可。

凑合看个图：



本来字的位置在外围的横线处，我们希望到内部的横线位置，需要调节水平和垂直的偏移；水平和垂直的平移方向为绿色的箭头；大概就这样。

4、绘制图像
/**
	 * 绘制图片
	 * 
	 * @param startAngle
	 * @param sweepAngle
	 * @param i
	 */
	private void drawIcon(float startAngle, int i)
	{
		// 设置图片的宽度为直径的1/8
		int imgWidth = mRadius / 8;
 
		float angle = (float) ((30 + startAngle) * (Math.PI / 180));
 
		int x = (int) (mCenter + mRadius / 2 / 2 * Math.cos(angle));
		int y = (int) (mCenter + mRadius / 2 / 2 * Math.sin(angle));
 
		// 确定绘制图片的位置
		Rect rect = new Rect(x - imgWidth / 2, y - imgWidth / 2, x + imgWidth
				/ 2, y + imgWidth / 2);
 
		mCanvas.drawBitmap(mImgsBitmap[i], null, rect, null);
 
	}

绘制图片主要就是图片的中心的确定，这里我们固定图片大小为直径的1/8；至于圆心的确定，看下图：
我们需要图片的中心，为每个块块的中间：



我们希望图片在中间的那个点，点距离圆心即center的距离为r = mRadius /2 / 2 ;

绿线与水平线的夹角为a = 360 / count / 2 ，本图为30 ；

于是那个点的坐标为：(mCenter + r * cos a , mCenter + r * sina );

其他的点同理，唯一变化就是a 的角度 ，在计算时需要把a转化为弧度制。

 集合图和上面的代码好好理解下。

到此基本我们的圆盘就绘制好了。



5、让圆盘先滚一会
怎么让圆盘滚动呢？如果你足够细心，应该发现我们的draw里面有这么一句：

mStartAngle += mSpeed;

其实每次draw都会让mStartAngle += mSpeed;看起来就是滚动了。

那么滚动，其实就是去设置mSpeed即可。

嗯，是的，如果单纯想滚动，只要去设置mSpeed就行了；但是，这样就行了么，就拿我们这个奖项来说，你敢1/6的概率拿到大奖么，你个IT公司让人抽到妹子一只咋办。

所以我们还要来控制用户抽奖的概率，这里我们让用户中奖的产品在开始滚的时候就决定了。是不是玩转盘的时候很傻很天真，以为可以中大奖。

/**
	 * 点击开始旋转
	 * 
	 * @param luckyIndex
	 */
	public void luckyStart(int luckyIndex)
	{
		// 每项角度大小
		float angle = (float) (360 / mItemCount);
		// 中奖角度范围（因为指针向上，所以水平第一项旋转到指针指向，需要旋转210-270；）
		float from = 270 - (luckyIndex + 1) * angle;
		float to = from + angle;
		// 停下来时旋转的距离
		float targetFrom = 4 * 360 + from;
		/**
		 * <pre>
		 *  (v1 + 0) * (v1+1) / 2 = target ;
		 *  v1*v1 + v1 - 2target = 0 ;
		 *  v1=-1+(1*1 + 8 *1 * target)/2;
		 * </pre>
		 */
		float v1 = (float) (Math.sqrt(1 * 1 + 8 * 1 * targetFrom) - 1) / 2;
		float targetTo = 4 * 360 + to;
		float v2 = (float) (Math.sqrt(1 * 1 + 8 * 1 * targetTo) - 1) / 2;
 
		mSpeed = (float) (v1 + Math.random() * (v2 - v1));
		isShouldEnd = false;
	}

当外部调用luckyStart即可以旋转，index为停下来的位置，水平位置开始算，即0为相机，1为IPAD。
这里又开始牵扯数学了：

float from = 270 - (luckyIndex + 1) * angle;
		float to = from + angle;

这个from , to 比较简单，就是确定中将范围，比如我index=0，则只要转了210-270之间，我们的相机都会被垂直向上的指针指向。
那么这个targetFrom是干嘛的，是决定你点击停止的时候转多长距离，这里我们设置为4圈多一点，这个多一点就是上面的from和to。

最麻烦就是v1的计算了，既然我们希望决定停下里的位置，那么这个速度就是我们去计算出来的，怎么算呢？

我们旋转的距离有了targetFrom，然后我们点击的时候mSpeed -= 1;也就是说速度是递减的，每次减去1。

递减说明是个等差数列，等差数列的和是targetFrom。

等差数列的求和公式大家记得否：（首项+末项）*（项数）/ 2

我们的首项是v1 ，末项肯定是0 ， 项数 （v1/ 1 + 1）加个1为向上进一位。 

那么式子就是： （v1 + 0 ) * (v1 / 1 +1) /2 = targetFrom ; 只有v1是未知数，一元二次方程的解，大家还记得否，不记得我来写 : 



于是我们的v1就是v1=-1+(1*1 + 8 *1 * target)/2;

好了，尼玛求出来v1，为啥我们代码还有个v2，这是因为v1停下来永远在某个块块的边界，我们屌丝又不傻，你每次停一个位置，都知道你造假。

那么我们就求个v2，这个停下块块的最后位置。

最后我们的速度为v1，v2间的一个随机数，也就是在某个块块中间任意位置。这样就可以让你觉得每次都在这个块块，但是指针位置还不同。

好了，这里就是最复杂的地方了，如果你比较善良，不想内置这个功能，那就随便设置个速度吧。

6、让圆盘停止滚动
别忘了，我们5计算那么多，都是从水平那个距离为0开始计算的，于是我们的停止代码是这样的：

public void luckyEnd()
	{
		mStartAngle = 0;
		isShouldEnd = true;
	}

最后贴出我们的主布局文件和Activity
7、布局文件和MainActivity
package com.zhy.demo_zhy_06_choujiangzhuanpan;
 
import android.app.Activity;
import android.os.Bundle;
import android.view.View;
import android.view.View.OnClickListener;
import android.widget.ImageView;
 
import com.zhy.view.LuckyPanView;
 
public class MainActivity extends Activity
{
	private LuckyPanView mLuckyPanView;
	private ImageView mStartBtn;
 
	@Override
	protected void onCreate(Bundle savedInstanceState)
	{
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
 
		mLuckyPanView = (LuckyPanView) findViewById(R.id.id_luckypan);
		mStartBtn = (ImageView) findViewById(R.id.id_start_btn);
 
		mStartBtn.setOnClickListener(new OnClickListener()
		{
			@Override
			public void onClick(View v)
			{
				if (!mLuckyPanView.isStart())
				{
					mStartBtn.setImageResource(R.drawable.stop);
					mLuckyPanView.luckyStart(1);
				} else
				{
					if (!mLuckyPanView.isShouldEnd())
 
					{
						mStartBtn.setImageResource(R.drawable.start);
						mLuckyPanView.luckyEnd();
					}
				}
			}
		});
	}
 
}

<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:background="#ffffff"
    android:layout_height="match_parent" >
 
    <com.zhy.view.LuckyPanView
        android:id="@+id/id_luckypan"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent"
        android:layout_centerInParent="true"
        android:padding="30dp" />
    
    <ImageView 
        android:id="@+id/id_start_btn"
        android:src="@drawable/start"
        android:layout_centerInParent="true"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        />
 
 
</RelativeLayout>

终于写完了，数学把我这类渣渣计算的不行不行的。ps：抠图真恶心，爱歌撒时候给我传递些艺术的造诣和ps的技术呢。。。。
好了，我们的按钮是用布局文件加上的，方便大家自己定制按钮~~~并且大家的奖项，颜色，以及图片可以自己定义，这个不用说了吧，修改count，以及那几个数组就行。

有可能的话，还会写一篇SurfaceView做游戏的博文，不过案例可能会在网上进行寻找，哈。


--------------------- 
作者：鸿洋_ 
来源：CSDN 
原文：https://blog.csdn.net/lmj623565791/article/details/41722441 
版权声明：本文为博主原创文章，转载请附上博文链接！
