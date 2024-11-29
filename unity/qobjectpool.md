# 对象池

基础基于UnityEngine.Pool也可以更改其他基础对象池或者自己实现 

## 简单类型对象池
```csharp
    /// <summary>
    /// 基础类型对象池
    /// </summary>
    public static class QObjectPool<T> where T : class, new() {
        private static ObjectPool<T> Instance;
        static QObjectPool() {
            if (typeof(T).Is(typeof(IQPoolObject))) {
                Instance = new ObjectPool<T>(() => new T(), obj => (obj as IQPoolObject).OnPoolGet(), obj => (obj as IQPoolObject).OnPoolRelease());
            }
            else {
                Instance = new ObjectPool<T>(() => new T());
            }
        }
        public static T Get() {
            return Instance.Get();
        }
        public static PooledObject<T> Get(out T obj) {
            return Instance.Get(out obj);
        }
        public static void Release(T obj) {
            Instance.Release(obj);
        }
        public static void Release<TKey>(Dictionary<TKey, T> dic) {
            foreach (var item in dic) {
                Release(item.Value);
            }
            dic.Clear();
        }
        public static void Release(List<T> list) {
            foreach (var item in list) {
                Release(item);
            }
            list.Clear();
        }
    }
```
对象池接口
```csharp
    public interface IQPoolObject {
        void OnPoolGet();
        void OnPoolRelease();
    }
```
## GameObject对象池
```csharp
    /// <summary>
    /// GameObject对象池
    /// </summary>
    public static class QGameObjectPool{
        private static Dictionary<GameObject, ObjectPool<GameObject>> GameObjectPools = new Dictionary<GameObject, ObjectPool<GameObject>>();
        internal static ObjectPool<GameObject> GetPool(GameObject prefab, Func<GameObject> createFunc, Action<GameObject> OnGet = null, Action<GameObject> actionOnRelease = null, Action<GameObject> actionOnDestroy = null, int maxSize = 10000) {
            if (!GameObjectPools.TryGetValue(prefab, out var pool)) {
                if (createFunc == null) {
                    throw new Exception(("不存在对象池[" + prefab + "]" + typeof(GameObject)));
                }
                Action<GameObject> OnRelease = actionOnRelease;
                OnGet += obj => {
                    foreach (var poolObj in obj.GetComponents<IQPoolObject>()) {
                        poolObj.OnPoolGet();
                    }
                };
                OnRelease += obj => {
                    foreach (var poolObj in obj.GetComponents<IQPoolObject>()) {
                        poolObj.OnPoolRelease();
                    }
                };
                pool = new ObjectPool<GameObject>(createFunc, OnGet, OnRelease, actionOnDestroy, true, 10, maxSize);
                GameObjectPools[prefab] = pool;
            }
            return pool;
        }
        public static ObjectPool<GameObject> GetPool(GameObject prefab, int maxSize = 1000) {
            return GetPool(prefab, () => {
                var result = UnityEngine.Object.Instantiate(prefab);
                result.name = prefab.name;
                result.GetComponent<QPoolObject>(true).prefab = prefab;
                return result;
            }, obj => {
                //obj.transform.SetParent(null, true);
                obj.transform.localScale = prefab.transform.localScale;
                obj.transform.localPosition = prefab.transform.localPosition;
                obj.transform.localRotation = prefab.transform.localRotation;
                var rect = prefab.transform.RectTransform();
                if (rect != null) {
                    var newRect = obj.transform.RectTransform();
                    newRect.anchorMin = rect.anchorMin;
                    newRect.anchorMax = rect.anchorMax;
                    newRect.anchoredPosition = rect.anchoredPosition;
                }
                obj.SetActive(true);
            },
            obj => {
                obj.SetActive(false);
                //obj.transform.SetParent(Instance.transform, true);
            }, obj => {
                GameObject.Destroy(obj);
            }, maxSize);
        }
        public static GameObject Get(GameObject prefab, Transform parent = null) {
            if (prefab == null)
                return null;
            var pool = GetPool(prefab);
            var obj = pool.Get();
            while (obj == null) {
                obj = pool.Get();
            }
            if (parent != null) {
                if (obj.transform.parent != parent) {
                    obj.transform.SetParent(parent, true);
                }
                obj.transform.localPosition = Vector3.zero;
                obj.transform.rotation = prefab.transform.rotation;
            }
            return obj;
        }
        public static GameObject Get(GameObject prefab, Vector3 position, Quaternion rotation = default) {
            var obj = Get(prefab);
            if (position != default) {
                obj.transform.position = position;
            }
            if (rotation != default) {
                obj.transform.rotation = rotation;
            }
            return obj;
        }
        public static void Release(GameObject gameObject) {
            if (gameObject == null)
                return;
            var tag = gameObject.GetComponent<QPoolObject>();
            try {
                if (tag != null && GameObjectPools.TryGetValue(tag.prefab, out var pool)) {
                    pool.Release(gameObject);
                    return;
                }
            }
            catch (Exception e) {
                QDebug.LogError("回收[" + tag?.prefab?.transform?.GetPath() + "]【" + gameObject + "】出错 " + e);
                return;
            }
            UnityEngine.Object.Destroy(gameObject);
        }
        public static void PoolRelease(this GameObject poolObj) {
            Release(poolObj);
        }
        public static void PoolReleaseList(this IList<GameObject> poolObjs) {
            foreach (var item in poolObjs) {
                item.PoolRelease();
            }
            poolObjs.Clear();
        }
    }
```
对象池附加类
```csharp
	/// <summary>
	/// GameObject对象池Get时自动添加 
	/// </summary>
	public class QPoolObject : MonoBehaviour, IQPoolObject
	{
		[QName("对象池"), QReadonly, SerializeField]
		internal GameObject prefab =null;
		[QName("延迟自动回收")]
		public float delayRelease = -1; 
		public UnityEvent OnGet = new UnityEvent();
		public UnityEvent OnRelease = new UnityEvent();
		private bool isReleased = false;
		void IQPoolObject.OnPoolGet()
		{
			isReleased = false;
			if (delayRelease > 0)
			{
				StartCoroutine(DelayRelease());
			}
			QTool.DelayInvoke(OnGet.Invoke);
		}
		void IQPoolObject.OnPoolRelease() {
			isReleased = true;
			OnRelease.Invoke();
		}
		public void PoolRelease() {
			gameObject.PoolRelease();
		}
		private IEnumerator DelayRelease()
		{
			yield return new WaitForSecondsRealtime(delayRelease);
			if (!isReleased)
			{
				gameObject.PoolRelease();
			}
		}
	}
```