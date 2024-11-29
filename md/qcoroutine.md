# 协程逻辑

自己实现的协程逻辑 不依赖于Unity 可以手动更新
> 最开始是为帧同步实现的调用顺序可控的协程 虽然挺易用的 但后来还是被我从库中删除了 因为协程遍历的GC无法避免
```csharp
    public static class QCoroutine {
		private static QDelayList<IEnumerator> List { set; get; } = new QDelayList<IEnumerator>();
		public static bool IsRunning(this IEnumerator enumerator) {
			if (List.Contains(enumerator)) {
				return true;
			}
			return false;
		}
		public static IEnumerator Wait(Func<bool> func) {
			while (!func()) {
				yield return null;
			}
		}
		public static IEnumerator WaitCountZero<T>(this IList<T> List) {
			yield return Wait(() => List.Count == 0);
		}
		public static IEnumerator Start(this IEnumerator enumerator) {
			//QDebug.LogError("Start " + enumerator);
			if (UpdateIEnumerator(enumerator)) {
				List.Add(enumerator);
			}
			//else
			//{
			//	QDebug.LogError("Stop " + enumerator);
			//}
			return enumerator;
		}
		public const int RunImmediateMaxLoopTimes = 100;
		/// <summary>
		/// 以同步方式运行一次更新
		/// </summary>
		public static void RunImmediate(this IEnumerator enumerator) {
			for (int i = 0; i < RunImmediateMaxLoopTimes; i++) {
				if (!UpdateImmediate(enumerator)) {
					return;
				}
			}
			QDebug.LogError(enumerator + " 立即运行次数超过 " + RunImmediateMaxLoopTimes + " 次循环");
		}
		public static IEnumerator Start(this IEnumerator enumerator, List<IEnumerator> coroutineList) {
			IEnumerator ie = null;
			ie = enumerator.OnCallBack(() => coroutineList.Remove(ie));
			coroutineList.Add(ie);
			return ie.Start();
		}
		public static IEnumerator OnCallBack(this IEnumerator enumerator, Action callBack) {
			yield return enumerator;
			callBack?.Invoke();
		}
		public static void Stop(this IEnumerator enumerator) {
			List.Remove(enumerator);
			enumerator.RunImmediate();
		}
		static Dictionary<YieldInstruction, float> YieldInstructionList = new Dictionary<YieldInstruction, float>();
		/// <summary>
		/// 处理Unity内置的等待逻辑
		/// </summary>
		private static bool UpdateIEnumerator(YieldInstruction yieldInstruction) {
			if (yieldInstruction is WaitForSeconds waitForSeconds) {
				var m_Seconds = (float)waitForSeconds.GetValue("m_Seconds");
				if (!YieldInstructionList.ContainsKey(yieldInstruction)) {
					YieldInstructionList.Add(yieldInstruction, Time.time);
				}
				if (Time.time > YieldInstructionList[yieldInstruction] + m_Seconds) {
					YieldInstructionList.Remove(yieldInstruction);
					return false;
				}
				else {
					return true;
				}
			}
			else {
				Debug.LogError(nameof(QCoroutine) + "暂不支持 " + yieldInstruction + "\n" + QSerializeType.Get(yieldInstruction.GetType()));
				return false;
			}
		}
		private static bool UpdateImmediate(this IEnumerator enumerator) {
			var start = enumerator.Current;
			if (enumerator.Current is YieldInstruction ie) {
				//跳过处理YieldInstruction
			}
			else if (enumerator.Current is IEnumerator nextChild) {
				if (UpdateImmediate(nextChild)) {
					return true;
				}
			}
			enumerator.MoveNext();
			if (enumerator.Current != null && start != enumerator.Current) {
				return UpdateImmediate(enumerator);
			}
			else {
				return false;
			}
		}
		/// <summary>
		/// 更新迭代
		/// </summary>
		/// <param name="enumerator">迭代器</param>
		/// <returns>true继续等待 false时结束等待</returns>
		private static bool UpdateIEnumerator(IEnumerator enumerator) {
			var start = enumerator.Current;
			if (enumerator.Current is YieldInstruction ie) {
				if (UpdateIEnumerator(ie)) {
					return true;
				}
			}
			else if (enumerator.Current is IEnumerator nextChild) {
				if (UpdateIEnumerator(nextChild)) {
					return true;
				}
			}
			var result = enumerator.MoveNext();
			if (enumerator.Current != null && start != enumerator.Current) {
				return UpdateIEnumerator(enumerator);
			}
			else {
				return result;
			}
		}
		public static void Update() {
			List.Update();
			List.RemoveAll(ie => !UpdateIEnumerator(ie));
		}
		public static void StopAll() {
			List.Clear();
		}
	}
	public class QCoroutineQueue<T> {
		private Func<T, IEnumerator> action;
		private Queue<T> Queue { get; set; } = new Queue<T>();
		public int Count => Queue.Count;
		public bool IsRunning => Count > 0;
		public T Current => IsRunning ? Queue.Peek() : default;
		public QCoroutineQueue(Func<T, IEnumerator> action) {
			this.action = action;
		}
		private void Run(T value) {
			action(value).OnCallBack(() => {
				if (value.Equals(Current)) {
					Queue.Dequeue();
				}
				if (Queue.Count > 0) {
					Run(Queue.Peek());
				}
			}).Start();
		}
		public IEnumerator Start(T value) {
			Queue.Enqueue(value);
			if (Queue.Count == 1) {
				Run(value);
			}
			while (Queue.Count > 0) {
				yield return null;
			}
		}
		public void Clear() {
			Queue.Clear();
		}
	}
```

*    今天开始将会陆续更新一些shader文章与视频教程，给与对于学习shader朋友一些思路与参考，个人学习也是参考网上各路大神的文章，所以各位想要学习也最好多看看文章，视频虽然更快学会但有时会少一些思考，相辅相成更容易学习新的知识。  
    
*   首先，我们先通过后期效果来了解shader的工作原理，我的教程都是通过直接编写使用之后再慢慢消化内在的原理，所以我们先把shader想象成一个最简单的图像处理程序。
    
*   模糊效果作为最常用的后期效果之一，其中的均值模糊是最好实现的一种模糊，他的原理非常简单就是将一个像素点的颜色和它周围的像素点取平均值。
    
 
![](https://article.biliimg.com/bfs/article/99610692c35472bab118f3ab3b5b43f040077cd0.png)
 
红色框内数值与周围进行均值运算
 
*    我们知道计算机中常用颜色组合为RGB与RGBA，R G B 即为 红色 绿色 蓝色 的英文缩写，通过这三原色合成我们看到的多种色彩，而A则为阿尔法的英文缩写，而阿尔法主要是用来表示颜色的透明度。
    
*   这三种颜色常用0-255的整数或者0-1小数来表示他们的大小，那我们通过计算一点的像素值时和他周围数值进行平均即可，当所有像素点都进行此操作时就完成了均值模糊的效果。
    
 
![](https://article.biliimg.com/bfs/article/08cc0e3304dd2f232815f770825dae6a292f1799.png)
 
全部代码
 
*   写shader会有很多重复的代码，不过我们不要慌，我们就修改了下图中frag这一个函数内的内容，其中代码也非常简单，书写方式也和C与C#等很像，注意其中有一些使用的变量是上面定义的，其他代码则为固定框架，多用几次就习惯了。
    
 
![](https://article.biliimg.com/bfs/article/24d2d22df657a3a83230301234b2201b3aca80d7.png)
 
片段着色器函数
 
*   注意一下Properties这里面定义的都是对外公开的变量，在材质球上可以在监视视窗直接赋值如下
    
 
![](https://article.biliimg.com/bfs/article/8de4a6011fb1174300b42730054d77769739f39e.png)
 
Properties
 
*   我们可以简单更改一下此处代码对模糊半径变量进行对外公开
    
 
![](https://article.biliimg.com/bfs/article/c035109f07264680cd89c76bb4c0ac25de5be99b.png)
 
模糊半径变量
 
  
 
![](https://article.biliimg.com/bfs/article/08e14573e290653d019b54860e234cb767a08cf2.png)
 
更改后的Properties
 
*   将shader赋值到一个新建的材质球上 同时将材质球给与一个plane给与一张贴图并调节BlurRaius变量并查看效果
    
 
  
 
![](https://article.biliimg.com/bfs/article/f5c9eba1594c18ef5c652fad673fc8d562417009.png)
 
BlurRadius变量
 
  
 
![](https://article.biliimg.com/bfs/article/97c94cace75156a7ee1e42a180ba6bcc6da91936.png)
 
  
BlurRadius变量为0时即为不模糊  
 
  
 
![](https://article.biliimg.com/bfs/article/ed5d72e14163b19a0f2f2cb0972887cf0d270415.png)
 
  
BlurRadius变量为3时
 
  
 
  
 
  
 
![](https://article.biliimg.com/bfs/article/932c5c25bfc088f082221bc2f0e05b1f2fafee8d.png)
 
BlurRadius变量为20时图片失真
 
*   如此，我们简单的模糊效果就完成了，是不是道理很简单，实现也非常简单，但其现在是作用于物体而不是相机，如果想对相机起作用就要用到C#脚本中事件函数
    
 
                OnRenderImage(RenderTexture src, RenderTexture dest)
 
*   有此函数的脚本必须挂在Camera下才会被调用 src为源图像 dest为目标图像（输出图像），了解这些我们来看一下C#代码
    
 
![](https://article.biliimg.com/bfs/article/9ebadde331da3c82743a6995474b772fd182ba64.png)
 
AverageBlur类
 
  
 
*   可以看到我们用此函数来处理摄像机的后期效果，Blit可以指定材质球效果，此处的material为父物类创建，代码如下，主要是方便以后书写其他摄像机效果
    
 
![](https://article.biliimg.com/bfs/article/2e1f2cff8d2a7c06dda283c160552d6b19434a8b.png)
 
父类 ScreenEffect类
 
*   将脚本拖到摄像机下并赋值对应效果shader我们就得到一个摄像机后期效果了
    
 
![](https://article.biliimg.com/bfs/article/39086f12a9213517c5ae2d5560c4eb41c5be2453.png)
 
AverageBlur
 
*   通过调节对应属属性我们可以得到不同的模糊效果迭代次数的增多可以得到更好的效果，当然开销也会百年的非常大，如下
    
 
![](https://article.biliimg.com/bfs/article/dd459a21669fa502c423403f5c665a030a94096b.png)
 
0-2.6-5
 
  
 
![](https://article.biliimg.com/bfs/article/088cab90930315a575565cf4845e829c582773f2.png)
 
降采样
 
*   提高将采样次数可以大大减少开销，同时帮助我们获得更好的模糊效果，因为降低分辨率本来就是一种低成本的模糊效果。如下所示
    
 
![](https://article.biliimg.com/bfs/article/54a41bb1e5fcf47015958b15a4f978dbbe470f24.png)
 
2-2.4-2
 
*   好的本篇文章到这里就接近尾声了，项目源码地址为
    
*   https://github.com/QinZhuo/ShaderLab/tree/master/Assets/ScreenEffect/Blur
    
*   之后会陆续更新其他shader效果
    
 
  
 
本文章参考文章
 
https://blog.csdn.net/puppet_master/article/details/52547442  
 
https://blog.csdn.net/poem_qianmo/article/details/51871531