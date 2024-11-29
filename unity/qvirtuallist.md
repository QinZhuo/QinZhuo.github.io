# 虚拟列表

对[动态列表](qobjectlist)的功能动态拓展 用于对于UI列表的性能优化 只有在显示范围内的UI才会创建赋值
```csharp
	[RequireComponent(typeof(QObjectList))]
	public class QVirtualList : MonoBehaviour {
		public QObjectList list;
		public ScrollRect scrollRect;
		public LayoutGroup layoutGroup;
		public float ViewportUp{ get; private set; }
		public float ViewportDown { get;private set; }
		private void Reset() {
			list = GetComponent<QObjectList>();
			scrollRect = GetComponentInParent<ScrollRect>();
			layoutGroup = GetComponentInParent<LayoutGroup>();
		}
		private void UpdateViewportSize() {
			ViewportUp = scrollRect.viewport.Up();
			ViewportDown = scrollRect.viewport.Down();
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
			list.DelayClear();
			if(layoutGroup is VerticalLayoutGroup vertical) {
				vertical.padding.top = 0;
				vertical.padding.bottom = 0;
				for (int i = 0; i < VirtualCount; i++) {
					Debug.LogError($"{i} {!(GetDown(i) > scrollRect.viewport.Up()|| GetUp(i) < scrollRect.viewport.Down())} \n {GetDown(i) > scrollRect.viewport.Up()} {GetUp(i) < scrollRect.viewport.Down()} {scrollRect.viewport.up} " +
						$" \n {GetDown(i)} > {ViewportUp}"+
						$"\n {GetUp(i)} < {ViewportDown}"  );

					if (GetDown(i) > ViewportUp) {
						vertical.padding.top += (int)(list.prefab.transform as RectTransform).Height();
					}
					else if (GetUp(i) < ViewportDown) {

						vertical.padding.bottom -= (int)(list.prefab.transform as RectTransform).Height();
					}
					else {
						GetVirtual(i);
					}
				}
			}
		}
		public float GetDown(int index) {
			var rect = list.prefab.transform as RectTransform;
			return rect.Down() - index * rect.Height();
		}
		public float GetUp(int index) {
			var rect = list.prefab.transform as RectTransform;
			return rect.Up() - index * rect.Height();
		}
		private void Start() {
			scrollRect.onValueChanged.AddListener(v2 => {
				Fresh();
			});
		}
		private void Update() {
			UpdateViewportSize();
		}
	}
```

