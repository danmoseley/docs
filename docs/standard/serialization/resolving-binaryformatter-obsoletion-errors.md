---
title: "Resolving BinaryFormatter obsoletion and disablement errors"
description: This article describes resolving compile-time warnings caused by calling obsolete BinaryFormatter APIs or runtime errors caused by invoking BinaryFormatter within web apps.
ms.date: "07/20/2020"
ms.author: levib
author: GrabYourPitchforks
---

# Resolving BinaryFormatter obsoletion and disablement errors

This document serves as the landing page for __https://aka.ms/binaryformatter__. Error messages containing this URL are shown to developers in one of two situations:

- At compile time, a .cs file contained a call to the obsolete `BinaryFormatter.Serialize` or `BinaryFormatter.Deserialize` methods; or
- At runtime, somebody invoked `BinaryFormatter.Serialize` or `BinaryFormatter.Deserialize` but the feature was disabled.

This document is targeted toward users who receive these error messages and gives guidance for how to respond to them in your code.

For further reading on why `BinaryFormatter` APIs are being marked obsolete, see the [`BinaryFormatter` security guide](https://docs.microsoft.com/dotnet/standard/serialization/binaryformatter-security-guide) and [long-term obsoletion strategy](https://aka.ms/binaryformatter/obsoletion).

## Obsoletion of BinaryFormatter APIs

__For code that targets .NET 5.0 or later__, the methods `BinaryFormatter.Serialize` and `BinaryFormatter.Deserialize` are marked obsolete (as warning). These warnings are observable at compile time. If the project is compiled with "warnings as errors" enabled, these warnings will cause the project's build to fail.

__For code that targets earlier versions of .NET or netstandard__, the methods `BinaryFormatter.Serialize` and `BinaryFormatter.Deserialize` are not marked obsolete. No warnings will be observed at compile time.

In all cases, these checks are strictly compile-time checks. The calls to the obsolete APIs will succeed at runtime, even on .NET 5.0, unless `BinaryFormatter` functionality has been disabled at runtime. See the section titled _BinaryFormatter disabled by default in some projects_ below for more information on this.

### Recommended resolution

We strongly recommend that developers follow the recommendations in the [`BinaryFormatter` security guide](https://docs.microsoft.com/dotnet/standard/serialization/binaryformatter-security-guide) and stop using `BinaryFormatter` in their code bases.

The security guide provides recommendations for how to choose alternative serializers like _System.Text.Json_'s `JsonSerializer` or _System.Xml.Serialization_'s `XmlSerializer`.

The below sample code demonstrates converting an existing `BinaryFormatter` call site to use `JsonSerializer`.

```cs
/*
 * !! DANGEROUS, NOT RECOMMENDED !!
 * Demonstrates calls to BinaryFormatter Serialize & Deserialize.
 */

using System;
using System.IO;
using System.Runtime.Serialization.Formatters.Binary;

void UseBinaryFormatter_WriteToDisk(PurchaseOrder order)
{
    // Save the purchase order to disk
    using (FileStream writeStream = new FileStream("myfile.bin", FileMode.Create))
    {
        BinaryFormatter formatter = new BinaryFormatter();
        formatter.Serialize(writeStream, order);
    }
}

PurchaseOrder UseBinaryFormatter_ReadFromDisk()
{
    // Now read the purchase order back from disk
    using (FileStream readStream = new FileStream("myfile.bin", FileMode.Open))
    {
        BinaryFormatter formatter = new BinaryFormatter();
        return (PurchaseOrder)formatter.Deserialize(readStream);
    }
}

/*
 * !! SAFE, RECOMMENDED !!
 * Demonstrates calls to JsonSerializer Serialize and Deserialize.
 */

using System;
using System.IO;
using System.Text.Json;

// !! USES SAFE, RECOMMENDED APIS !!
void UseJsonSerializer_WriteToDisk(PurchaseOrder order)
{
    string serializedData = JsonSerializer.Serialize(order);
    File.WriteAllText("myfile.txt", serializedData);
}

// !! USES SAFE, RECOMMENDED APIS !!
PurchaseOrder UseJsonSerializer_ReadFromDisk()
{
    string serializedData = File.ReadAllText("myfile.txt");
    return JsonSerializer.Deserialize<PurchaseOrder>(serializedData);
}
```

### Alternative resolution: suppressing the warning

> We strongly recommend removing use of `BinaryFormatter` throughout your code base. If you suppress these warnings, you should do so only after a thorough risk assessment of your application. Suppressing these warnings should only be a temporary solution while you work on a long-term plan to remove `BinaryFormatter` call sites from your code base. See the [`BinaryFormatter` security guide](https://docs.microsoft.com/dotnet/standard/serialization/binaryformatter-security-guide) and [long-term obsoletion strategy](https://aka.ms/binaryformatter/obsoletion) for more information.

The warning code associated with `BinaryFormatter` obsoletions is __`SYSLIB0011`__. The easiest way to suppress the warnings is to surround the individual call site with `#pragma` directives, as shown below.

```cs
PurchaseOrder UseBinaryFormatter_ReadFromDisk()
{
    // Now read the purchase order back from disk
    using (FileStream readStream = new FileStream("myfile.bin", FileMode.Open))
    {
        BinaryFormatter formatter = new BinaryFormatter();
#pragma warning disable SYSLIB0011
        return (PurchaseOrder)formatter.Deserialize(readStream);
#pragma warning restore SYSLIB0011
    }
}
```

The warning can also be suppressed within the .csproj, as shown below.

```xml
<PropertyGroup>
  <OutputType>Exe</OutputType>
  <TargetFramework>net5.0</TargetFramework>
  <!-- Disable "BinaryFormatter is obsolete" warnings for entire project -->
  <NoWarn>$(NoWarn);SYSLIB0011</NoWarn>
</PropertyGroup>
```

If the warning is suppressed in the .csproj, it will remain suppressed for all .cs files within the project. The `SYSLIB0011` warning code is specific to `BinaryFormatter`. Suppressing `SYSLIB0011` will not suppress warnings caused by using other obsolete APIs.

## BinaryFormatter disabled by default in some projects

### ASP.NET 5.0 disables BinaryFormatter by default

__Starting with ASP.NET 5.0__, `BinaryFormatter.Serialize` and `BinaryFormatter.Deserialize` are __disabled by default__. This means that calls to these APIs will fail at runtime with `NotSupportedException`. Because this is a runtime check, it is independent from the obsoletion warnings mentioned earlier. The runtime check will trigger regardless of whether the developer suppressed the `SYSLIB0011` warning in their code.

To address this issue, see the earlier section titled __Recommended resolution__ for sample code demonstrating removing `BinaryFormatter` from your code.

If you wish to continue using `BinaryFormatter` within your ASP.NET 5.0+ application, you must modify the project's .csproj file to re-enable `BinaryFormatter` functionality. The sample below shows how to do this.

> Warning: use of `BinaryFormatter` is a leading source of security vulnerabilities in web applications. We strongly recommend __against__ re-enabling support for `BinaryFormatter` in web applications. See the [`BinaryFormatter` security guide](https://docs.microsoft.com/dotnet/standard/serialization/binaryformatter-security-guide) for more information.

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net5.0</TargetFramework>
    <!-- Warning: setting the below switch is *NOT* recommended in web apps -->
    <EnableUnsafeBinaryFormatterSerialization>true</EnableUnsafeBinaryFormatterSerialization>
  </PropertyGroup>

</Project>
```

> There is no supported mechanism for individual libraries to re-enable `BinaryFormatter` code paths within an ASP.NET 5.0+ application. If a web app wishes to re-enable `BinaryFormatter`, it must be done via the web app's own .csproj file. Programmatic attempts to re-enable `BinaryFormatter` are unsupported and risk destabilizing the runtime.

### wasm (Blazor Client) 5.0 disables BinaryFormatter

__Starting with .NET 5.0 wasm apps__, `BinaryFormatter.Serialize` and `BinaryFormatter.Deserialize` are __hardcoded disabled__. There is no mechanism to re-enable `BinaryFormatter` functionality in these application types.

`JsonSerializer` is the preferred serializer for these application types. See the earlier section titled __Recommended resolution__ for sample code demonstrating usage of `JsonSerializer. See [the full _System.Text.Json_ user guide](https://docs.microsoft.com/dotnet/standard/serialization/system-text-json-how-to) for advanced samples and more information.

### Projects which voluntarily disable BinaryFormatter

By default, other project types (such as console apps) allow use of `BinaryFormatter`. These apps may choose to forbid `BinaryFormatter` code paths from running, perhaps as part of a larger threat abatement strategy.

To forbid use of `BinaryFormatter` within an application, set the switch in your .csproj as shown below.

```xml
<PropertyGroup>
  <OutputType>Exe</OutputType>
  <TargetFramework>net5.0</TargetFramework>
    <!-- Disallow use of BinaryFormatter application-wide. -->
    <EnableUnsafeBinaryFormatterSerialization>false</EnableUnsafeBinaryFormatterSerialization>
</PropertyGroup>
```

This can also be set through an app's _runtimeconfig.template.json_. An example of this is provided below.

```json
{
  "configProperties": {
    "System.Runtime.Serialization.EnableUnsafeBinaryFormatterSerialization": false
  }
}
```

See [".NET Core run-time configuration settings"](https://docs.microsoft.com/dotnet/core/run-time-config/) for more information on JSON-based configuration.

> If an app uses one of the two switches above to disable `BinaryFormatter`, there is no supported mechanism for individual libraries to re-enable it. Programmatic attempts to re-enable `BinaryFormatter` are unsupported and risk destabilizing the runtime.