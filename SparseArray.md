## SparseArray

#### 官方说明

SparseArray 是整数到对象的映射，不是普通的对象数组，索引中可以存在空白。相比 HashMap（ Integers to Objects ），它可以提高内存的使用效率，因为可以避免 Keys 的自动装箱并且数据结构不依赖额外结构体。

此容器的映射使用数组数据结构，Keys 使用二分查找。这样的实现不适合大量数据。它通常比传统的 HashMap 要慢，因为查找使用二分搜索，添加、删除实体在数组中。对于存储几百个数据来说，性能差异不明显，小于50%。

为了提高性能，在移除键的时候有一个优化：不是立刻压缩数组，而是将需要移除的实体标记为已删除。这个实体可以被相同的键重新使用，或者在之后的垃圾回收过程中统一删除。垃圾回收可能在数组需要增加或者实体值被检索的任何时间发生。

#### 构造方法

- 默认构造方法：默认容量10.
- 传入容量构造方法。

#### 特有方法

- index 是 SpareArray 的特有属性，SpareArray 内部使用两个数组存储 Keys 和 Values。

- ``` int[] mKeys
  private int[] mKeys;
  private Object[] mValues;
  ```


- indexOfKey(int key);
- indexOfValue(int key);
- keyAt(int index);
- valueAt(int index);
- setValueAt(int index);
- removeAt(int index);

#### put 方法

```
public void put(int key, E value) {
	int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

	if (i >= 0) {
		mValues[i] = value;
	} else {
		i = ~i;

		if (i < mSize && mValues[i] == DELETED) {
			mKeys[i] = key;
			mValues[i] = value;
			return;
		}

		if (mGarbage && mSize >= mKeys.length) {
			gc();

			// Search again because indices may have changed.
			i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
		}

		mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
		mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
		mSize++;
	}
}
```

1. 通过二分查找 key 对应的 index。
2. 如果 index >= 0，说明数组中有此 key，直接覆盖原值。
3. 否则 index < 0，如果 index < mSzie，并且 mValue[index] 被标记为删除，直接覆盖已删除的并返回。
4. 如果被标记为 gc，并且 mSize >= mKeys.length，通过 gc 回收需要删除的值，然后通过二分查找获取 index。
5. 否则不满足 gc，需要扩容，currentSize <= 4 ? 8 : currentSize * 2;
6. 在 index 位置上插入键与值，并且 mSize++。

#### remove 方法

```
public void remove(int key) {
	delete(key);
}

public void delete(int key) {
	int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

	if (i >= 0) {
		if (mValues[i] != DELETED) {
			mValues[i] = DELETED;
			mGarbage = true;
		}
	}
}
```

1. 通过二分查找 key 对应的 index。
2. 如果 index >= 0，标记 mValue[index] 已删除，同时标记 mGarbage 为待回收。

#### gc 方法

```
private void gc() {
	int n = mSize;
	int o = 0;
	int[] keys = mKeys;
	Object[] values = mValues;

	for (int i = 0; i < n; i++) {
		Object val = values[i];

		if (val != DELETED) {
			if (i != o) {
				keys[o] = keys[i];
				values[o] = val;
				values[i] = null;
			}

			o++;
		}
	}

	mGarbage = false;
	mSize = o;
}
```

#### 相关类

- SparseIntArray：int-int
- SparseBooleanArray：int-boolean
- SparseLongArray：int-long

### 与HashMap对比

##### 优点

- 避免了基本数据类型的装箱操作。
- 不需要额外的结构体，单个元素的存储成本更低。
- 数据量小的情况下，随机访问效率更低。

##### 缺点

- 插入操作可能需要复制数组，增删效率降低。
- 数据量巨大时，复制数组成本巨大，`gc()` 成本也巨大。
- 数据量巨大时，查询效率也会明显下降。