# 可搜索下拉框
基于AdvancedDropdown制作的可搜索下拉框
<!-- 
[](https://ncase.me/sight-and-light/draft7.html ':include') -->

<!-- [gist: script.js](https://raw.githubusercontent.com/QinZhuo/IDG_Game_One/refs/heads/master/Assets/Scripts/GunBase.cs ':include :type=code') -->


<!-- [](https://github.com/QinZhuo/IDG_Game_One/blob/master/Assets/Scripts/GunBase.cs ':include :type=code') -->

![可搜索下拉框](qdropdown.png)
```csharp
public class QAdvancedDropdown : AdvancedDropdown {
	public AdvancedDropdownItem Root { get; private set; }
	public QDictionary<string,AdvancedDropdownItem> Items { get; private set; }
	public QDictionary<AdvancedDropdownItem, Action> Actions { get; private set; } = new();
	public IMouseEvent MouseEvent { get; set; }
	public QAdvancedDropdown(string title, Action<string> onItemSelected = null) : base(new AdvancedDropdownState()) {
		Root = new AdvancedDropdownItem(title);
		Items = new(key => {
			if (key.Contains('/')) {
				var index = key.LastIndexOf('/');
				var start = key.Substring(0, index);
				var end = key.Substring(index+1);
				var item = new AdvancedDropdownItem(end);
				Items[start].AddChild(item);
				if (onItemSelected != null) {
					Actions[item] = () => onItemSelected(key);
				}
				return item;
			}
			else {
				var item = new AdvancedDropdownItem(key);
				Root.AddChild(item);
				if (onItemSelected != null) {
					Actions[item] = () => onItemSelected(key);
				}
				return item;
			}
		});

	}
	public void AddSeparator() {
		Root.AddSeparator();
	}
	public void Add(string key,Action action=null) {
		var item = Items[key];
		if (action != null) {
			Actions[item] = action;
		}
	}
	public void Add(IList<string> keys) {
		foreach (var key in keys) {
			Add(key);
		}
	}
	protected override AdvancedDropdownItem BuildRoot() {
		return Root;
	}
	protected override void ItemSelected(AdvancedDropdownItem item) {
		base.ItemSelected(item);
		Actions[item]?.Invoke();
	}
}
```

