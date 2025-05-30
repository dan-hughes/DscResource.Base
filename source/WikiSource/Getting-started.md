---
Category: Documentation
---

# Getting started

This module provides a base class for creating DSC resources.

Assuming you are using the DscCommunity Sampler template, add `DscResource.Base` to your `RequiredModules.psd1`.

Ensure that your `prefix.ps1` contains `using module .\Modules\DscResource.Base` to import the module.

Add the following into a new file (typical location `source\Classes`):

```powershell
[DscResource()]
class MyDscResource : ResourceBase
{
    <#
        PROPERTIES
    #>

    MyDscResource () : base ($PSScriptRoot)
    {
        $this.ExcludeDscProperties = @()
        $this.FeatureOptionalEnums = $false
    }

    # Required DSC Methods, these call the method in the base class.
    [MyDscResource] Get()
    {
        # Call the base method to return the properties.
        return ([ResourceBase] $this).Get()
    }

    [void] Set()
    {
        # Call the base method to enforce the properties.
        ([ResourceBase] $this).Set()
    }

    [System.Boolean] Test()
    {
        # Call the base method to test all of the properties that should be enforced.
        return ([ResourceBase] $this).Test()
    }

    <#
        Base method Get() call this method to get the current state as a Hashtable.
        The parameter properties contains any DSC properties defined as 'Key'.
    #>
    [System.Collections.Hashtable] GetCurrentState([System.Collections.Hashtable] $properties)
    {
    }

    <#
        Base method Set() call this method with the properties that should be
        enforced and that are not in desired state.
    #>
    hidden [void] Modify([System.Collections.Hashtable] $properties)
    {
    }

    # OPTIONAL
    <#
        This method can be overridden if resource specific property asserts are
        needed. The parameter properties will contain the properties that are
        assigned a value.
    #>
    hidden [void] AssertProperties([System.Collections.Hashtable] $properties)
    {
    }

    <#
        This method can be overridden if resource specific property normalization
        is needed. The parameter properties will contain the properties that are
        assigned a value.
    #>
    hidden [void] NormalizeProperties([System.Collections.Hashtable] $properties)
    {
    }
}
```

## Properties

Value type properties like `Boolean`, `Int`, `UInt` must have their type made
nullable e.g. `[Nullable[System.Boolean]]`.

Reference type properties like `String` are already nullable.

Enums may be used, by default only for properties marked `Key` or `Mandatory`.
There is an optional feature flag available that allows use of Enums for `Optional`
properties without a default value configured.
See [Constructor Properties](#constructor-properties).

### Ensure

The base class ships with an `Ensure` Enum which you may use. This is typically
a mandatory property. If optional it must be set to a default value
e.g. `$Ensure = [Ensure]:Present`.

```powershell
[DscProperty(Mandatory)]
[Ensure]
$Ensure
```

### Reasons

The base class will populate the `Reasons` property if the actual state does not
match the current state. If a `Reasons` property is not provided then this is omitted.

To utilize `Reasons` this you must create your own `Reason` class for the module.
This needs to be unique among all the loaded classes (e.g resources) used on the
same target computer.

This is typically is prefixed with the module name. The example below is what is
used in `SqlServerDsc`.

```powershell
<#
    .SYNOPSIS
        The reason a property of a DSC resource is not in desired state.

    .DESCRIPTION
        A DSC resource can have a read-only property `Reasons` that the compliance
        part of Azure AutoManage Machine Configuration uses.
        The property Reasons holds an array of SqlReason. Each SqlReason
        explains why a property of a DSC resource is not in desired state.
#>

class SqlReason
{
    [DscProperty()]
    [System.String]
    $Code

    [DscProperty()]
    [System.String]
    $Phrase
}
```

This is how you would declare the property.

```powershell
[DscProperty(NotConfigurable)]
[SqlReason[]]
$Reasons
```

## Constructor

The constructor above shows the minimum implementation required.
`$PSScriptRoot` is passed down to the base class to update the location of the
localization files. Without this you may see an error when using the resource of
not being able to find localization data. How to use [String Localization](#localization).

### Constructor Properties

The following properties can be set in the constructor.

#### ExcludeDscProperties

By default all `Key`, `Mandatory` and `Optional` DSC properties are enforced in
the base class.
`ExcludeDscProperties` is used to prevent the provided properties from being enforced.
This is in the format of a list of strings.

```powershell
 MyDscResource () : base ($PSScriptRoot)
    {
        $this.ExcludeDscProperties = @(
            'PropertyOne',
            'PropertyTwo'
        )

        $this.FeatureOptionalEnums = $false
    }
```

#### FeatureOptionalEnums

`FeatureOptionalEnums` when enabled will allow the use of Enums for optional
properties where a default value is not set.

These do require special consideration when a default value is not provided,
the Enum must be created with a starting value of 1. Leaving 0 for uninitialized.
This is due to not being able to declare the type as `Nullable` in PowerShell DSC.

```powershell
enum MyEnum
{
    Square = 1
    Circle
}

[DscResource()]
class MyDscResource : ResourceBase
{

    [DscProperty()]
    [MyEnum]
    $MyProperty

    MyDscResource () : base ($PSScriptRoot)
    {
        $this.ExcludeDscProperties = @(
            'PropertyOne',
            'PropertyTwo'
        )

        $this.FeatureOptionalEnums = $true
    }
}
```

## Methods

The two methods you must implement which are `GetCurrentState()` and `Modify()`.
There are two optional methods which you may use these are `AssertProperties()` and
`NormalizeProperties()`.

### GetCurrentState

This method is used to get the current state of the resource based on the passed
key properties. This is called by `Get()`.

The `$properties` argument passed into this method is a hashtable of any
properties marked `Key`.

These properties should normally be used for any arguments on commands called
within this method `$properties.PropertyName` and not using `$this.PropertyName`.

The hashable only needs to return the values that are not `$null`. Any that are
`$null` can be omitted as these are added to the hashtable from the calling
method.

### Modify

This method is used to change the state of the resource being controlled.
This is called by `Set()`.

The `$properties` passed into this method are the ones that need to be enforced and
are not in the desired state.

This function can either be used for any `Set/Add/Remove` behavior, but if this
needs breaking up then additional methods can be created within your class
and called from here.

### AssertProperties

This method is called before getting any state or resources and is used to assert
or validate properties or property sets as this is not available in class-based
DSC resources.

Example uses:

- Checking use of mutually exclusive properties.
- Validating values are correct e.g. StartTime < EndTime.

### NormalizeProperties

This method is used to Normalize any property values that may have been provided
by the user but can be standardized.

Example uses:

- Formatting a file path
- Formatting a URL

## Localization

String localization is supported in the base class. This allows the use
of a localization variable in the format `$this.localizedData.MyLocalizedStringKey`
in code and then this be referenced in a `.psd1` file for each language translation.

By default the `en-US` culture must be populated (location `source\en-US`).
The filename must be in the format `ClassName.strings.psd1`. If no strings are
declared then an empty file must be configured.

Example empty strings file.

```powershell
<#
    .SYNOPSIS
        The localized resource strings in English (en-US) for the
        class {ClassName}.
#>

ConvertFrom-StringData @'
'@
```

Example populated strings file.

```powershell
<#
    .SYNOPSIS
        The localized resource strings in English (en-US) for the
        class {ClassName}.
#>

ConvertFrom-StringData @'
    MyLocalizedStringKey = This is a localized string with a passed in value '{0}'. (MD0001)
'@
```
