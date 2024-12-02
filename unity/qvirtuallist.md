# 虚拟列表

对[动态列表](unity/qobjectlist)的功能动态拓展 用于对于UI列表的性能优化 只有在显示范围内的UI才会创建赋值

对于上千上万个可滚动UI拥有极大的优化效果

```csharp
using System;
using UnityEngine;
using UnityEngine.UI;
namespace QTool {
	[RequireComponent(typeof(QObjectList))]
	public class QVirtualList : MonoBehaviour {
		public QObjectList list;
		public ScrollRect scrollRect;
		public LayoutGroup layoutGroup;
		private void Reset() {
			list = GetComponent<QObjectList>();
			scrollRect = GetComponentInParent<ScrollRect>();
			layoutGroup = GetComponentInParent<LayoutGroup>();
		}
		public int VirtualCount { get; private set; }
		private event Action<int> GetVirtual;
		private bool dirty = false;
		public void SetVirtualData(int VirtualCount, Action<int> GetVirtual) {
			list.DelayClear();
			this.VirtualCount = VirtualCount;
			this.GetVirtual = GetVirtual;
			dirty = true;
		}
		private void Start() {
			scrollRect.onValueChanged.AddListener(v2 => {
				dirty = true;
			});
		}
		private void LateUpdate() {
			if (dirty && scrollRect.viewport.Width() > 0) {
				dirty = false;
				Fresh();
			}
		}
		public void Fresh() {
			list.DelayClear();
			switch (layoutGroup) {
				case VerticalLayoutGroup vertical: {
					vertical.padding.top = 0;
					vertical.padding.bottom = 0;
					var viewportStart = scrollRect.viewport.Up();
					var viewportEnd = scrollRect.viewport.Down();
					var layoutStart = (layoutGroup.transform as RectTransform).Up();
					var prefabSize = (int)(list.prefab.transform as RectTransform).Height();
					for (int i = 0; i < VirtualCount; i++) {
						var start = layoutStart - i * prefabSize;
						var end = layoutStart - (i + 1) * prefabSize;
						if (end > viewportStart) {
							vertical.padding.top += prefabSize;
						}
						else if (start < viewportEnd) {

							vertical.padding.bottom += prefabSize;
						}
						else {
							GetVirtual(i);
						}
					}
				}
				break;
				case HorizontalLayoutGroup horizontal: {
					horizontal.padding.left = 0;
					horizontal.padding.right = 0;
					var viewportStart = scrollRect.viewport.Left();
					var viewportEnd = scrollRect.viewport.Right();
					var layoutStart = (layoutGroup.transform as RectTransform).Left();
					var prefabSize = (int)(list.prefab.transform as RectTransform).Width();
					for (int i = 0; i < VirtualCount; i++) {
						var start = layoutStart + i * prefabSize;
						var end = layoutStart + (i + 1) * prefabSize;
						if (end < viewportStart) {
							horizontal.padding.left += prefabSize;
						}
						else if (start > viewportEnd) {

							horizontal.padding.right += prefabSize;
						}
						else {
							GetVirtual(i);
						}
					}
				}
				break;
				default:
					break;
			}
		}
	}
}
```

