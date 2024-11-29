# 计时器

可循环使用的计时器实现
> 手动调用更新 会保留上次调用余下的时间间隔
```csharp
   [Serializable]
	public class QTimer
	{
		[SerializeField]
		private float _time = 1;
		[SerializeField,QReadonly]
		private float _curTime = 0;
		public float Time { get => _time; }
		public float CurTime { get => _curTime; }
		public bool IsOver => CurTime >= Time;
		public QTimer()
		{

		}
		public QTimer(float Time, bool startOver = false)
		{
			Reset(Time, startOver);
		}
		public void Clear()
		{
			_curTime = 0;
		}
		public void Over()
		{
			_curTime = Time;
		}
		public void SetCurTime(float curTime)
		{
			_curTime = curTime;
		}
		public void Reset(float time, bool startOver = false)
		{
			_time = time;
			Clear();
			if (startOver) Over();
		}
		/// <summary>
		/// 更新deltaTime并检测计时是否结束
		/// </summary>
		/// <param name="deltaTime">更新数值</param>
		/// <param name="autoClear">计时成功是否清空</param>
		/// <returns>计时时否成功</returns>
		public bool Check(float deltaTime = 0, bool autoClear = true)
		{
			_curTime = CurTime + deltaTime;
			var timeOffset = CurTime - Time;
			if (timeOffset >= 0)
			{
				if (autoClear)
				{
					_curTime = timeOffset;
				}
				else
				{
					_curTime = Time;
				}
				return true;
			}
			else
			{
				return false;
			}
		}

	}
```