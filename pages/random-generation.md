# Random generation

When using the random generation, if the loop is particularly fast the system tick may not be different and the same number is generated twice, a small delay may help

## Random number generator

```csharp
Random r = new Random();

public void Start() {
    // Create a random integer value
	Int variableRandom = r.Next(min, max);
    // Create a random double value
	Double variableRandom = r.NextDouble()
}
```

## Random strings generator

### Using custom character list

```csharp
private static Random random = new Random();

public static string RandomString(int stringLength) 
{
    // Set the subset of characters to be used
    const string usableCharacters = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
    // Return the random string
    return new string(Enumerable.Repeat(usableCharacters, stringLength)
        .Select(s => s[random.Next(s.Length)]).ToArray());
}

public void Start() {
    // Call the method to generate the string
	String randomString = RandomString(10);
}
```

### Using random GUID

```csharp
private string RandomString()
{
    // Create a new guid
    Guid newGuid = Guid.NewGuid();
    // Remove dashes to keep only alphanumeric characters
    return newGuid.ToString().Replace("-", "");
}
```
