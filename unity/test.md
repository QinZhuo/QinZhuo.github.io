```csharp
using Hono.Core;
using Hono;
using QTool.UI;
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;
using System.Reflection;
namespace QTool {
	/// <summary>
	/// 选择确认页面
	/// </summary>
	public abstract class SelectPanel<T, TItem> : QUI<T> where T : SelectPanel<T, TItem> where TItem : class {
		public QObjectList items;
		[UnityEngine.Serialization.FormerlySerializedAs("next")]
		public Button confirm;
		public Button cancel;
		public TItem current;
		private Func<TItem, bool> filter;
		protected Func<TItem, bool> typeFilter;

		public MonoBehaviour itemViewComp;
		private IView<TItem> ItemView => itemViewComp as IView<TItem>;

		private Action<TItem> _callBack;

		public abstract void FreshList();
		public virtual void CheckView(TItem item) {
			if (filter?.Invoke(item) == false)
				return;
			if (typeFilter?.Invoke(item) == false)
				return;
			var toggle = items.AddView(item).GetComponentInChildren<Toggle>();
			toggle.onValueChanged.SetListener(isOn => {
				if (isOn) {
					current = item;
					OnSelect();
				}
				else if (current == item) {
					current = null;
				}
			});
			toggle.isOn = item == current;
		}
		protected virtual void OnSelect() {
			ItemView?.SetData(current);
		}
		private static bool CheckNull() {
			if (Instance.items.Count == 0) {
				if (Instance.typeFilter != null) {
					var cache = Instance.typeFilter;
					Instance.typeFilter = null;
					Instance.items.Clear();
					Instance.FreshList();
					Instance.typeFilter = cache;
					if (Instance.items.Count > 0) {
						return false;
					}
				}
				return true;
			}
			return false;
		}
		
		public static void ShowPanel(Action<TItem> callBack, Func<TItem, bool> filter, string nullMsg = null, TItem defaultItem = default) {
			Instance._callBack = callBack;
			Instance.current = defaultItem;
			Instance.filter = filter;
			Instance.items.DelayClearAndFreshToggles();
			Instance.FreshList();
			if (!nullMsg.IsNull()) {
				if (CheckNull()) {
					TipPanel.ShowMsg(nullMsg);
					HidePanel();
					return;
				}
			}
			if (Instance.confirm != null)
				Instance.confirm.onClick.SetListener(Instance.OnConfirmBtnClicked);
			Instance.cancel?.onClick.SetListener(() => {
				callBack?.Invoke(null);
				HidePanel();
			});
			ShowPanel();
		}
		public override void OnFresh() {
			base.OnFresh();
			items.DelayClearAndFreshToggles();
			FreshList();
			ItemView?.SetData(current);
		}

		public void OnConfirmBtnClicked() {
			_callBack?.Invoke(current);
			HidePanel();
		}
	}
}
```