# Variables formatting

## Zero padding

Reads value of `i`, converts into a string and add a zero to the left if value is less than 10

```csharp
text = i.ToString("D2");
```

## Print the hex equivalent of any character

```csharp
keyPressed = "a";
Log.Info(BitConverter.ToString(Encoding.Default.GetBytes(keyPressed)));
```
