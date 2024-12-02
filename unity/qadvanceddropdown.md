# 可搜索下拉框
基于AdvancedDropdown制作的可搜索下拉框
![alt text](img/dropdown.png)
```csharp
public class QAdvancedDropdown : AdvancedDropdown {
	public AdvancedDropdownItem Root { get; private set; }
	public QDictionary<string,AdvancedDropdownItem> Items { get; private set; }
	private event Action<string> onItemSelected;
	public QAdvancedDropdown(string title, Action<string> onItemSelected) : base(new AdvancedDropdownState()) {
		Root = new AdvancedDropdownItem(title);
		Items = new(key => {
			if (key.SplitTowString("/", out var start, out var end)) {
				var item = new AdvancedDropdownItem(key);
				Items[start].AddChild(item);
				return item;
			}
			else {
				var item = new AdvancedDropdownItem(key);
				Root.AddChild(item);
				return item;
			}
		});
		this.onItemSelected = onItemSelected;
	}
	public void Add(string key) {
		_ = Items[key];
	}
	public void Add(params string[] keys) {
		foreach (var key in keys) {
			Add(key);
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
		onItemSelected?.Invoke(item.name);
	}
}
```

