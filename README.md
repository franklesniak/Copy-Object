# Copy-Object

This PowerShell function performs an intelligent "deep clone" of an object by using serialization/deserialization techniques.

## Motivation

If you run the following code:

```PowerShell
$SourceObject = @([AppDomain]::CurrentDomain.GetAssemblies())
$NewObject = $SourceObject
```

You might expect that you've created a copy of $SourceObject and stored it in $NewObject.
However, if you start making changes to $NewObject, you will see that $SourceObject has also been affected!

This is because PowerShell does not "deep-clone" an object when the equal sign operator is used - instead, it performs a "shallow copy."
In effect, this means that the two variables share the same object.

## Usage

`$intReturnCode = Copy-Object ([ref]$DestinationObject) ([ref]$SourceObject) [$intCopyDepth] [$boolSourceObjectSafe]`

As illustrated, two, three, or four positional arguments are required.

The first positional argument is required.
It must be a reference to a variable into which the object will be cloned/copied

The second positional argument is also required.
It must be a reference to the source variable from which the object will be cloned/copied.

The third positional argument is optional.
If supplied, it must be a positive integer that specifies the copy depth.
If omitted or set to `$null`, a copy depth of 2 will be used.

The fourth positional argument is optional.
If supplied, it must be a boolean value that indicates whether the variable supplied in the second argument should be considered "safe", i.e., generated from a trusted process without any possibility of interference from a threat actor.
See the security vulnerability note, below.

The function returns an integer that indicates the success/failure of the clone/copy operation:

- A return code of `0` indicates that the object was cloned successfully and fully - i.e., that the source object was marked as serializable, the function caller indicated that it is "safe", and the function was successfully able to use BinaryFormatter to clone the object.

- A return code of `1` indicates that the object was cloned successfully, but only up to the specified copy depth.
Therefore, it may not be a full replica of the source object.

- A return code of `2` indicates a failure - the object was not cloned successfully.

### Example 1

```PowerShell
$SourceObject = @([AppDomain]::CurrentDomain.GetAssemblies())
$DestinationObject = $null
$intReturnCode = Copy-Object ([ref]$DestinationObject) ([ref]$SourceObject) 1
```

This example clones `SourceObject` into `$DestinationObject` at a copy depth of 1, meaning that properties of the properties of $SourceObject would not be copied, if applicable.

### Example 2

```PowerShell
$SourceObject = @([AppDomain]::CurrentDomain.GetAssemblies())
$DestinationObject = $null
$intReturnCode = Copy-Object ([ref]$DestinationObject) ([ref]$SourceObject) 3
```

This example performs a copy "more deeply" than the previous example.
The copy depth of three means that three levels of recursion would occur to copy properties of the properties of the properties of $SourceObject.

### Example 3

```PowerShell
$SourceObject = New-Object 'System.Collections.Generic.List[System.String]'
for ($intCounter = 1; $intCounter -le 10000; $intCounter++) {
    $SourceObject.Add('Item' + ([string]$intCounter))
}
$DestinationObject = $null
$intReturnCode = Copy-Object ([ref]$DestinationObject) ([ref]$SourceObject) 3 $true
```

This example clones `$SourceObject`, an object that is marked as serializable, into `$DestinationObject`.
Because the fourth parameter was set to `$true`, a BinaryFormatter clone/copy operation is performed, and we would expect that `$DestinationObject` is a full clone of `$SourceObject`.

## Notes

### Security Vulnerabilities

If the fourth positional parameter is set to `$true`, and if the source object is marked as serializable, then `BinaryFormatter` will be used to clone the object.
However, `BinaryFormatter` has inherent security vulnerabilities that Microsoft has stated to be unfixable.
Therefore, Microsoft actively discourages the use of `BinaryFormatter`.

Nevertheless, when the source object is generated from a trusted process and its input is not supplied by an external system or otherwise not something that a threat actor can manipulate, then it can be desirable to use BinaryFormatter to clone an object.
This is because `BinaryFormatter` performs a bit-by-bit clone of the source object, which is not only fast, but also guarantees that the new object is a full clone of the source object.

On the other hand, if the fourth positional parameter is omitted, set to `$false`, or set to `$null`, then the function will use "safe" serialization/deserialization techniques (JSON/XML), thereby avoiding the security vulnerability. However, it does so at the expense of potentially not cloning an object "deeply" enough, which means the new object may not be a full replica of the source object.

For more information on the security vulnerabilities inherent in the use of BinaryFormatter, review [Microsoft's documentation](https://learn.microsoft.com/en-us/dotnet/standard/serialization/binaryformatter-security-guide).

### Compatibility

This function is backward-compatible to PowerShell v1.0 and forward-compatible to the newest release.

To ensure backward compatibility, the function uses special error-handling techniques, does not use comment-based help, uses positional arguments instead of parameters, and will fall back to using slower object-cloning techniques (`Export-Clixml` / `Import-Clixml`) when necessary.
