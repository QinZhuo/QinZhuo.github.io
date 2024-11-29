# 虚拟列表

对[动态列表](qobjectlist)的功能动态拓展 用于对于UI列表的性能优化 只有在显示范围内的UI才会创建赋值
```csharp
   [RequireComponent(typeof(QObjectList))]
	public class QVirtualList : MonoBehaviour {
	public QObjectList list;
	private void Reset() {
		list = GetComponent<QObjectList>();
	}
	public int VirtualCount { get; private set; }
	private event Action<int> GetVirtual;
	public void SetVirtualData(int VirtualCount, Action<int> GetVirtual) {
		list.DelayClear();
		this.VirtualCount = VirtualCount;
		this.GetVirtual = GetVirtual;
		Fresh();
	}
	public void Fresh() {
		for (int i = 0; i < VirtualCount; i++) {
			GetVirtual(i);
		}
	}
}
```

