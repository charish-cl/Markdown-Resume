# 无限滚动工具

1. 初始化Item，通过上边距，间隔 Item大小预估要生成多少元素，初始化content的长度， 初始化三个数组，保存GameObject的数组，报错position 和 size的数组
2. 每帧判断当前视野 view中的最大索引，然后从后往前遍历，之前的元素全部显示，后面全部隐藏
3. 在上一步基础上，更新元素每帧位置，复用GameObject的数组

![image-20240719102320160](C:\Users\ForgotElk\AppData\Roaming\Typora\typora-user-images\image-20240719102320160.png)

## 核心代码

```cs
private int GetMaxIndexInView()
{
    float size = isVertical
        ? (_content.anchoredPosition.y + _viewSize.y)
        : (-_content.anchoredPosition.x + _viewSize.x);

    for (int i = 0; i < _positions.Count; i++)
    {
        if ((isVertical && size <= -_positions[i].y) || (!isVertical && size <= _positions[i].x))
        {
            return i;
        }
    }

    return _positions.Count - 1;
}
```

```cs
private void UpdateItemData(Vector2 value)
{
    if (_items == null) return;

    int visibleMaxIdx = GetMaxIndexInView();
    //i对应对象池，j对应数据
    for (int i = _items.Length - 1, j = visibleMaxIdx; i >= 0; j--, i--)
    {
        if (j < 0)
        {
            _items[i].SetActive(false);
            continue;
        }

        if (!_items[i].activeSelf)
            _items[i].SetActive(true);

        if (UpdateDataFunc != null)
        {
            UpdateDataFunc(j, _items[i]);
        }

        TempItems[j] = _items[i];
        _rects[i].anchoredPosition = _positions[j];
    }
}
```

# 资源替换工具

主要针对与Texture

1. 合并资源 把选中文件夹相似的资源给替换掉
2. 若有图片资源改动，选中一对一替换
3. 获取引用资源的对象
4. 清楚没有被依赖的资源

![image-20240719101145112](C:\Users\ForgotElk\AppData\Roaming\Typora\typora-user-images\image-20240719101145112.png)



## 如何判断图片相似？

比较像素，但不用每个一一比较，大致比较检查下就行了 

```cs
private static string GetImageKey(Texture2D image)
{
    Stopwatch stopwatch = new Stopwatch();
    stopwatch.Start();
    Color32[] pixels = image.GetPixels32();

    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < pixels.Length; i += 100) //  every 100th pixel 平均10ms
    {
        Color32 pixel = pixels[i];
        sb.Append(pixel.r);
        sb.Append(pixel.g);
        sb.Append(pixel.b);
        sb.Append(pixel.a);
    }
    stopwatch.Stop();
    UnityEngine.Debug.Log("Execution time: " + stopwatch.ElapsedMilliseconds + "ms");
    return sb.ToString();
}
```

## 如何获取引用？

.prefab ， .material 文件含有Guid





### 替换资源

把旧的Guid替换成新的Guid

```cs
      	/// <summary>
        /// 查找  并   替换 
        /// </summary>
        /// <param name="_oldGuid"></param>
        /// <param name="_newGuid"></param>
        private static void Find(string _oldGuid, string _newGuid)
        {
            if (fileContents == null || fileContents.Count == 0)
            {
                ReadFileContents();
            }

            List<string> keysToUpdate = new List<string>();

            //用Parallel加速，不在这里读写，只保存需要修改的文件
            Parallel.ForEach(fileContents, kvp =>
            {
                //文件中是否包含旧的Guid
                if (Regex.IsMatch(kvp.Value, _oldGuid))
                {
                    Debug.Log("替换了资源的路径：" + kvp.Key + GetRelativeAssetsPath(kvp.Key));
                    keysToUpdate.Add(kvp.Key); // 记录需要更改的 key
                }
                else
                {
                    Debug.Log("查看了的路径：" + kvp.Key);
                }
            });

            foreach (var key in keysToUpdate)
            {
                var newContent = fileContents[key].Replace(_oldGuid, _newGuid);
                fileContents[key] = newContent; // 统一替换
            }

            foreach (var key in keysToUpdate.Distinct()) // 使用 Distinct 方法确保相同的文件路径只写入一次
            {
                File.WriteAllText(key, fileContents[key]);
            }


            // 在这里进行文件写入等操作
            Debug.Log("替换结束");
            
            AssetDatabase.Refresh();
            
        }
```





# 打包工具

通用接口，不同平台实现不同实现

分为本地打包与远程打包

1. 本地打包 用作测试，用本地的资源服务器
2. 远程打包 供jenkins使用，

![image-20240719111027208](C:\Users\ForgotElk\AppData\Roaming\Typora\typora-user-images\image-20240719111027208.png)





# UI绑定工具

选中Ui类型，添加到绑定数据里生成代码

![image-20240719113403280](C:\Users\ForgotElk\AppData\Roaming\Typora\typora-user-images\image-20240719113403280.png)

![image-20240719113433874](C:\Users\ForgotElk\AppData\Roaming\Typora\typora-user-images\image-20240719113433874.png)