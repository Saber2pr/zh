在使用 Ant Design 的 Table 表格组件时，很多开发者会遇到一个非常让人困惑的问题：我明明设置了 `columns` 中某一列 `fixed: 'left'` 或 `fixed: 'right'`，为什么它**没生效**？

别急，本文就来帮你分析原因，并提供一个简单有效的解决方案。

---

## 🧩 问题复现

我们先来看一段典型的代码：

```tsx
<Table
  columns={[
    { title: '姓名', dataIndex: 'name', key: 'name', fixed: 'left' },
    { title: '年龄', dataIndex: 'age', key: 'age' },
    { title: '地址', dataIndex: 'address', key: 'address' },
  ]}
  dataSource={data}
/>
```

这样设置以后你可能会发现，“固定左侧的姓名列怎么并没有固定？”滚动的时候，它还是跟其他列一起滑动了。

---

## 🕵️ 原因分析

这是因为 Antd 的 Table 组件中，**固定列功能依赖于横向滚动（scroll.x）**。

简单来说：

> 如果表格没有设置 `scroll.x`，Antd 就不会启用横向滚动，也就不会启用固定列的逻辑。

也就是说，固定列想要生效，必须满足一个前提条件：**表格的宽度要足够宽，需要横向滚动。**

---

## ✅ 正确做法：添加 `scroll.x`

只需要给 Table 加上一个 `scroll` 属性并设置 `x` 值，问题立刻解决：

```tsx
<Table
  columns={[
    { title: '姓名', dataIndex: 'name', key: 'name', fixed: 'left' },
    { title: '年龄', dataIndex: 'age', key: 'age' },
    // 其他很多列...
    { title: '地址', dataIndex: 'address', key: 'address', fixed: 'right' },
  ]}
  dataSource={data}
  scroll={{ x: 1000 }} // 👈 关键就在这里
/>
```

### `scroll.x` 设置建议：

- ✅ 固定值，例如 `scroll={{ x: 1000 }}`；
- ✅ 使用 CSS 单位，例如 `scroll={{ x: '100%' }}` 或 `'max-content'`；
- ✅ 动态计算列总宽度再赋值，更加灵活；

---

## 🎯 总结

- 如果你使用了 `fixed: 'left'` 或 `fixed: 'right'`，但发现没效果；
- **大概率是因为你没设置 `scroll.x`！**
- 固定列必须依赖横向滚动才能生效，记得加上 `scroll={{ x: 某个值 }}`。

这个问题属于 Ant Design Table 的一个“使用前提”，知道之后其实很好解决，建议大家在使用固定列时**总是设置 `scroll.x`**，养成好习惯～
