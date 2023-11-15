# Random generation

When using the random generation, if the loop is particularly fast the system tick may not be different and the same number is generated twice, a small delay may help

## Random number generator

```csharp
Random r = new Random();

public void Start() {
	Int variableRandom = r.Next(min, max);
	Double variableRandom = r.NextDouble()
}
```

## Random strings generator

### Using custom character list

```csharp
private static Random random = new Random();

public static string RandomString(int length) {
        const string chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
        return new string(Enumerable.Repeat(chars, length)
            .Select(s => s[random.Next(s.Length)]).ToArray());
    }

public void Start() {
	String randStr = RandomString(10);
}
```

### Using random GUID

```csharp
Guid g = Guid.NewGuid();
string randomString = g.replace("-", "");
```
