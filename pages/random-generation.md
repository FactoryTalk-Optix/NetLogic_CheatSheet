# Random generation

When using the random generation, if the loop is particularly fast the system tick may not be different and the same number is generated twice, a small delay may help

## Random number generator

```csharp
Random r = new Random();

public void Start() 
{
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

public void Start() 
{
    // Call the method to generate the string
	String randomString = RandomString(10);
}
```

### Using random GUID

Please note, this should **never** be used to generate passwords or tokens as it's not random enough. If high security is required, use the [RandomNumberGenerator](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.randomnumbergenerator?view=net-8.0) instead.

```csharp
private string RandomString()
{
    // Create a new guid
    Guid newGuid = Guid.NewGuid();
    // Remove dashes to keep only alphanumeric characters
    return newGuid.ToString().Replace("-", "");
}
```

```csharp
private static string GenerateRandomString(int length = 8)
{
    const string alphanumericChars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";

    using var rng = System.Security.Cryptography.RandomNumberGenerator.Create();
    var bytes = new byte[length];
    rng.GetBytes(bytes);

    var chars = new char[length];
    for (int i = 0; i < length; i++)
    {
        chars[i] = alphanumericChars[bytes[i] % alphanumericChars.Length];
    }

    return new string(chars);
}
```

This one should be better

```csharp
[ExportMethod]
public static void GenerateRandomString(out string randomString)
{
    // Create a random string generator using the RandomNumberGenerator class
    using var rng = RandomNumberGenerator.Create();
    // Create a byte array to store the random values
    var randomBytes = new byte[32];
    // Fill the byte array with random values
    rng.GetBytes(randomBytes);
    // Convert the byte array to a string
    randomString = Convert.ToBase64String(randomBytes).Substring(0, 32);
}
```
