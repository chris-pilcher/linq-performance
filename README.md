# Linq Performance

This project benchmarks the performance of Linq in .NET 6.0 vs .NET 7.0. [Nick Chapsas: The INSANE performance boost of LINQ in .NET 7](https://www.youtube.com/watch?v=zCKwlgtVLnQ) is the inspiration for the project.

## Results

| Method  | Job      | Runtime  | Size |        Mean |     Error |    StdDev | Allocated |
| ------- | -------- | -------- | ---- | ----------: | --------: | --------: | --------: |
| Min     | .NET 6.0 | .NET 6.0 | 100  | 49,722.2 ns | 445.57 ns | 394.99 ns |      40 B |
| Max     | .NET 6.0 | .NET 6.0 | 100  | 44,944.5 ns | 797.90 ns | 707.31 ns |      40 B |
| Average | .NET 6.0 | .NET 6.0 | 100  | 47,587.6 ns | 814.22 ns | 836.15 ns |      40 B |
| Sum     | .NET 6.0 | .NET 6.0 | 100  | 44,973.7 ns | 898.94 ns | 999.17 ns |      40 B |
| Min     | .NET 7.0 | .NET 7.0 | 100  |    801.3 ns |  15.46 ns |  15.18 ns |         - |
| Max     | .NET 7.0 | .NET 7.0 | 100  |    795.9 ns |  15.34 ns |  17.67 ns |         - |
| Average | .NET 7.0 | .NET 7.0 | 100  |  1,029.1 ns |  11.25 ns |   9.40 ns |         - |
| Sum     | .NET 7.0 | .NET 7.0 | 100  |  2,982.8 ns |  52.84 ns |  49.43 ns |         - |

## Code Changes

### Before

```csharp
public static int Min(this IEnumerable<int> source)
{
    if (source == null)
    {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.source);
    }

    int value;
    using (IEnumerator<int> e = source.GetEnumerator())
    {
        if (!e.MoveNext())
        {
            ThrowHelper.ThrowNoElementsException();
        }

        value = e.Current;
        while (e.MoveNext())
        {
            int x = e.Current;
            if (x < value)
            {
                value = x;
            }
        }
    }

    return value;
}
```

### After

```csharp
public static int Min(this IEnumerable<int> source) => MinInteger(source);

private static T MinInteger<T>(this IEnumerable<T> source) where T : struct, IBinaryInteger<T>
{
    T value;

    if (source.TryGetSpan(out ReadOnlySpan<T> span))
    {
        if (span.IsEmpty)
        {
            ThrowHelper.ThrowNoElementsException();
        }

        // Vectorize the search if possible.
        int index;
        if (Vector.IsHardwareAccelerated && span.Length >= Vector<T>.Count * 2)
        {
            // The span is at least two vectors long. Create a vector from the first N elements,
            // and then repeatedly compare that against the next vector from the span.  At the end,
            // the resulting vector will contain the minimum values found, and we then need only
            // to find the min of those.
            var mins = new Vector<T>(span);
            index = Vector<T>.Count;
            do
            {
                mins = Vector.Min(mins, new Vector<T>(span.Slice(index)));
                index += Vector<T>.Count;
            }
            while (index + Vector<T>.Count <= span.Length);

            value = mins[0];
            for (int i = 1; i < Vector<T>.Count; i++)
            {
                if (mins[i] < value)
                {
                    value = mins[i];
                }
            }
        }
        else
        {
            value = span[0];
            index = 1;
        }

        // Iterate through the remaining elements, comparing against the min.
        for (int i = index; (uint)i < (uint)span.Length; i++)
        {
            if (span[i] < value)
            {
                value = span[i];
            }
        }

        return value;
    }

    using (IEnumerator<T> e = source.GetEnumerator())
    {
        if (!e.MoveNext())
        {
            ThrowHelper.ThrowNoElementsException();
        }

        value = e.Current;
        while (e.MoveNext())
        {
            T x = e.Current;
            if (x < value)
            {
                value = x;
            }
        }
    }

    return value;
}
```

