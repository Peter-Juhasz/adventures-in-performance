# Optimizing LINQ operators

The following writeup optimizes for:
- small element count
- basic collection types
- build time optimization
- single thread access


## Non-enumerated count
.NET already has a helper method called [TryGetNonEnumeratedCount](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Linq/src/System/Linq/Count.cs) to speed up scenarios like this, but it uses only runtime type matching to determine whether a count if available without enumeration:

```cs
public static int Count<T>(this IEnumerable<T> source)
{
	if (source.TryGetNonEnumeratedCount(int count))
	{
		return count;
	}

	// ...
}
```

But we can make this a build time optimization:

```cs
public static int Count<T>(this IReadOnlyCollection<T> source) => source.Count;
```


## Optimize for zero element count
Many operators can be optimized for zero element count:

```cs
public static bool Any<T>(this IReadOnlyCollection<T> source, Func<T, bool> predicate)
{
	if (source.Count == 0)
	{
		return false;
	}

	// ...
}
```

```cs
public static bool All<T>(this IReadOnlyCollection<T> source, Func<T, bool> predicate)
{
	if (source.Count == 0)
	{
		return true;
	}

	// ...
}
```

```cs
public static IEnumerable<T> Where<T>(this IReadOnlyCollection<T> source, Func<T, bool> predicate)
{
	if (source.Count == 0)
	{
		return Enumerable.Empty<T>();
	}

	// ...
}
```

```cs
public static IEnumerable<T> Distinct<T>(this IReadOnlyCollection<T> source, Func<T, bool> predicate)
{
	if (source.Count == 0)
	{
		return Enumerable.Empty<T>();
	}

	// ...
}
```

There are two options if `source` is empty, and these are the disadvantages:
- Return `source`, because it is empty anyway.
	- The underlying collection `source` may change after the operator and show different results for future calls.
	- The original collection `source` is passed to the outer caller and thus keeping a reference to that collection.
- Return `Enumerable.Empty<T>`.
	- Very first call for each `T` may take a little longer and allocate a long-term object on the heap.


## Optimize for single elements
In `Distinct`, does not matter what `comparer` we use, we can reuse the source collection if there is only a single element in it.

```cs
public static bool Distinct<T>(this IReadOnlyCollection<T> source, IEqualityComparer<T> comparer)
{
	if (source.Count <= 1)
	{
		return source;
	}

	// ...
}
```

The same applies to `OrderBy`, regardless of the `selector` or the `comparer`:

```cs
public static IEnumerable<T> OrderBy<T, TResult>(
	this IReadOnlyCollection<T> source,
	Func<T, TResult> selector,
	IEqualityComparer<TResult> comparer
)
{
	if (source.Count <= 1)
	{
		return source;
	}

	// ...
}
```


## Optimize enumeration for lists
For every enumeration on an `IEnumerable<T>`, we need to create an `IEnumerator<T>` to enumerate the elements. Since we work with interfaces here, this object is allocated on the heap. But in most cases, we deal with lists, which enumeration mechanism is known, so we can move that implementation upwards to spare a head allocation:

```cs
public static bool Any<T>(this IReadOnlyList<T> source, Func<T, bool> predicate)
{
	for (int i = 0; i < source.Count; i++)
	{
		if (predicate(source[i]))
		{
			return true;
		}
	}

	// ...
}
```

Note: this method looses the capability of *underlying collection changed* detection, so it can be used only in single thread scenarios.

You can find more information about value type enumerators and operators in this project: [valuelinq](https://github.com/Peter-Juhasz/valuelinq).

```cs
public static bool Any<T>(this IReadOnlyList<T> source, Func<T, bool> predicate)
{
	foreach (var item in source.AsValueEnumerable())
	{
		if (predicate(source[i]))
		{
			return true;
		}
	}

	// ...
}
```


## Optimize for small element count

```cs
public static IEnumerable<T> Where<T>(this IReadOnlyList<T> source, Func<T, bool> predicate)
{
	if (source.Count <= Threshold)
	{
		if (source.All(source, predicate))
		{
			return source;
		}
	}

	// ...
}
```

With `Distinct` on a list, if we expect duplications an edge case, with small element numbers we can check whether the element are distinct without allocating an enumerator, a hash set and a new storage for the results:

```cs
public static IEnumerable<T> Distinct<T>(this IReadOnlyList<T> source, IEqualityComparer<T> comparer)
{
	// ...

	if (source.Count <= Threshold)
	{
		for (int i = 0; i < source.Count; i++)
		for (int j = 0; j < source.Count; j++)
		{
			if (i == j)
			{
				continue;
			}

			if (!comparer.Equals(source[i], source[j]))
			{
				return Enumerable.Distinct(source, comparer);
			}
		}
	}

	return source;
}
```


## Optimize `First` and `Last` for lists
If the underlying collection is a list, we can use indexes instead of enumeration:

```cs
public static T First<T>(this IReadOnlyList<T> source) => source[0];
public static T Last<T>(this IReadOnlyList<T> source) => source[^1];
```

Sample applies for the other versions of these methods:

```cs
public static T? FirstOrDefault<T>(this IReadOnlyList<T> source) => source.Any() ? source[0] : default;
public static T? LastOrDefault<T>(this IReadOnlyList<T> source) => source.Any() ? source[^1] : default;
```


## Optimize `Take` and `Skip` for inbound and outbound ranges
If the source collection is in the range we can pass it through:

```cs
public static IEnumerable<T> Take<T>(this IReadOnlyCollection<T> source, int count)
{
	if (count == 0)
	{
		return Enumerable.Empty<T>();
	}

	if (source.Count <= count)
	{
		return source;
	}

	// ...
}
```

If we need to skip more than the source collection, it is an empty one:

```cs
public static IEnumerable<T> Skip<T>(this IReadOnlyCollection<T> source, int count)
{
	if (count == 0)
	{
		return Enumerable.Empty<T>();
	}

	if (count >= source.Count)
	{
		return Enumerable.Empty<T>();
	}

	// ...
}
```


## In-place transformations
If the underlying collection a transient one (e.g. a result of a previous LINQ operation), we can do some transformations inplace, instead of creating a new collection or enumeration:

```cs
public static IReadOnlyList<T> OrderByInplace<T>(this IReadOnlyList<T> source, IComparer<T> comparer)
{
	if (source is List<T> list)
	{
		list.Sort(comparer);
		return list;
	}

	// ...
}
```

```cs
public static IReadOnlyList<T> WhereInplace<T>(this IReadOnlyList<T> source, Func<T, bool> predicate)
{
	for (int i = 0; i < source.Count; i++)
	{
		var item = source[i];
		if (!predicate(item))
		{
			source.RemoveAt(i);
		}
	}

	return source;
}
```