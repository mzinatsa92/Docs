---
uid: config-builder
title: Configuration builders for ASP.NET
author: rick-anderson
description: Learn how to get configuration data from sources other than web.config
ms.author: riande
ms.date: 9/9/2018
ms.technology: aspnet
msc.type: content
---

<!-- Search for review for embedded review questions
https://docs.microsoft.com/en-us/dotnet/api/?view=netframework-4.7.2&term=KeyValueConfigBuilder
no results. Will this be published? `KeyValueConfigBuilder` abstract methods is needed to create a custom builder.

Should I PR your NuGet packages and change 
"A simple key/value Configuration Builder for the .Net Desktop Framework ..." to
"A simple key/value Configuration Builder for the .NET Framework .."
-->

By [Stephen Molloy](https://github.com/StephenMolloy) and [Rick Anderson](https://twitter.com/RickAndMSFT)

# Configuration builders for ASP.NET

Configuration builders provide a mechanism for ASP.NET apps to get configuration values. 

Configuration builders:

* Are available in .Net Framework .Net 4.7.1 and later.
* Provide a flexible mechanism to get started reading configuration values.
* Address some of the basic needs of apps as they move into a container and cloud focused environment.

## Key/value configuration builders

The most common usage of configuration builders is to provide a basic key/value replacement mechanism. Most of the `Microsoft.Configuration.ConfigurationBuilders` are basic key/value builders. Applications can also use the configuration builder concept to construct complex configuration dynamically.

## Key/value configuration builders settings

The following settings apply to all key/value configuration builders.

### Mode

The configuration builders use an external source of key/value information to populate selected key/value elements of the configuration system. Specifically, the `<appSettings/>` and `<connectionStrings/>` sections receive special treatment from the configuration builders. The builders work in three modes:

* `Strict` - The default mode. In this mode, the configuration builder will only operate on well-known key/value-centric configuration sections. `Strict` mode enumerates each key in the section. If a matching key is found in the external source:

   * The configuration builders replace the value in the resulting configuration section with the value from the external source.
* `Greedy` - This mode is closely related to `Strict` mode. Rather than being limited to keys that already exist in the original configuration:

  * The configuration builders dump all key/value pairs from the external source into the resulting configuration section.

* `Expand` - Operates on the raw XML before it's parsed into a configuration section object. It can be thought of as an expansion of tokens in a string. Any part of the raw XML string that matches the pattern `${token}` is a candidate for token expansion. If no corresponding value is found in the external source, then the token is not changed.

The following markup from *web.config* enables the [EnvironmentConfigBuilder](https://www.nuget.org/packages/Microsoft.Configuration.ConfigurationBuilders.Environment/) in `Strict` mode:

[!code-xml[Main](config-builder/MyConfigBuilders/WebDefault.config?name=snippet)]

The following code reads the `<appSettings/>` and `<connectionStrings/>` from the environment, if available:

[!code-csharp[Main](config-builder/MyConfigBuilders/About.aspx.cs)]

### Prefix handling

Key prefixes can simplify setting keys because:

* The .Net Framework configuration is complex and nested.
* External key/value sources are by nature basic and flat.

For example, use either approach to inject both `<appSettings/>` and `<connectionStrings/>` into the configuration via environment variables:

* With the `EnvironmentConfigBuilder` in the default `Strict` mode and the appropriate key names in the configuration file. The preceding code and markup takes this approach.
* Use two `EnvironmentConfigBuilder`s in `Greedy` mode with distinct prefixes. Using this approach the app can read `<appSettings/>` and `<connectionStrings/>` without needing to update the configuration file. For example:

[!code-xml[Main](config-builder/MyConfigBuilders/WebPrefix.config?name=snippet&highlight=11-99)]

With the preceding markup, the same flat key/value source can be used to populate configuration for two different sections.

<!-- Docs\aspnet\config-builder\MyConfigBuilders\Contact.aspx.cs doesn't work-->

### stripPrefix

`stripPrefix`: boolean, defaults to `false`. 

The preceding XML markup separates app settings from connection strings but requires all the keys to use the specified prefix. For example, "AppSetting_ServiceID". `stripPrefix` is used to remove the prefix added by the `prefix` attribute.

Applications typically strip off the prefix. The following markup strips the prefix:

[!code-xml[Main](config-builder/MyConfigBuilders/WebPrefixStrip.config?name=snippet&highlight=11-99)]

<!-- My sample doesn't work-->

### tokenPattern

`tokenPattern`: String, defaults to `@"\$\{(\w+)\}"`

The `Expand` behavior of the builders searches the raw XML for tokens that look like `${token}`. Searching is done with the default regular expression `@"\$\{(\w+)\}"`. The set of characters that matches `\w` is more strict than XML and many configuration sources allow. Use `tokenPattern` when more characters than `@"\$\{(\w+)\}"` are required in the token name.

`tokenPattern`: String:

* Allows developers to change the regex that is used for token matching.
* No validation is done to make sure it is a well-formed, non-dangerous regex.
* It must contain a capture group. The entire regex must match the entire token. The first capture must be the token name to look up in the configuration source.

## Configuration builders in Microsoft.Configuration.ConfigurationBuilders

### EnvironmentConfigBuilder

```xml
<add name="Environment"
    [mode|prefix|stripPrefix|tokenPattern] 
    type="Microsoft.Configuration.ConfigurationBuilders.EnvironmentConfigBuilder,
    Microsoft.Configuration.ConfigurationBuilders.Environment" />
```

The [EnvironmentConfigBuilder](https://www.nuget.org/packages/Microsoft.Configuration.ConfigurationBuilders.Environment/)`:

* Is the simplest of the configuration builders.
* Draws its values from the environment.
* Does not have any additional configuration options.
* The `name` attribute value is arbitrary.

**Note:** In a Windows container environment, variables set at run time are only injected into the EntryPoint process environment. Apps that run as a service or a non-EntryPoint process will not pick up these variables unless they are otherwise injected through a mechanism in the container. For [IIS](https://github.com/Microsoft/iis-docker/pull/41)/[ASP.Net](https://github.com/Microsoft/aspnet-docker)-based
 containers, the current version of [ServiceMonitor.exe](https://github.com/Microsoft/iis-docker/pull/41) handles this in the *DefaultAppPool* only. Other Windows-based container variants may need to develop their own injection mechanism for non-EntryPoint processes.

### UserSecretsConfigBuilder

```xml
<add name="UserSecrets"
    [mode|prefix|stripPrefix|tokenPattern]
    (userSecretsId="{secret string, typically a GUID}" | userSecretsFile="~\secrets.file")
    [optional="true"]
    type="Microsoft.Configuration.ConfigurationBuilders.UserSecretsConfigBuilder,
    Microsoft.Configuration.ConfigurationBuilders.UserSecrets" />
```

This configuration builder provides a feature similar to [ASP.NET Core Secret Manager](/aspnet/core/security/app-secrets).

The [UserSecretsConfigBuilder](https://www.nuget.org/packages/Microsoft.Configuration.ConfigurationBuilders.UserSecrets/) can be used in .NET Framework projects, but a secrets file must be specified. Alternatively, you can define the `UserSecretsId` property in the project file and create the raw secrets file in the correct location for reading. To keep external dependencies out of your project:

* The actual secret file is XML formatted. The XML formatting is an implementation detail, and the format should not be relied upon.
* If you need to share a *secrets.json* file with .NET Core projects, consider using the [SimpleJsonConfigBuilder(#simplejsonconfig). The `SimpleJsonConfigBuilder` format for .NET Core is an implementation detail subject to change.

Configuration attributes for `UserSecretsConfigBuilder`:

* `userSecretsId` - This is the preferred method for identifying an XML secrets file. It works similar to .Net Core, which uses a `UserSecretsId` project property to store this identifier. The string must be unique, it doesn't need to be a GUID. With this attribute, the `UserSecretsConfigBuilder` will look in a well-known local location (%APPDATA%\Microsoft\UserSecrets\&lt;userSecretsId&gt;\secrets.xml in Windows environments) for a secrets file belonging to this identifier.
* `userSecretsFile` - An optional attribute specifying the file containing the secrets. The `~` character can be used at the start to reference the application root. Either this attribute or the `userSecretsId` attribute is required. If both are specified, `userSecretsFile` takes precedence. <!-- review: | userSecretsFile="~\secrets.file" -  isn't ~/ a security risk? .NET Core secret manager stores values in %APPDATA%\Microsoft\UserSecrets so it won't be checked into github -->
* `optional`: boolean, default value `true` - Prevents an exception if the secrets file cannot be found. 
* The `name` attribute value is arbitrary.

The secrets file has the following format:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<root>
  <secrets ver="1.0">
    <secret name="secret key name" value="secret value" />
  </secrets>
</root>
```

### AzureKeyVaultConfigBuilder

```xml
<add name="AzureKeyVault"
    [mode|prefix|stripPrefix|tokenPattern]
    (vaultName="MyVaultName" |
     uri="https://MyVaultName.vault.azure.net")
    [connectionString="connection string"]
    [version="secrets version"]
    [preloadSecretNames="true"]
    type="Microsoft.Configuration.ConfigurationBuilders.AzureKeyVaultConfigBuilder,
    Microsoft.Configuration.ConfigurationBuilders.Azure" />
```

`AzureKeyVaultConfigBuilder` reads values stored in the [Azure Key Vault](/azure/key-vault/key-vault-whatis).

The `vaultName` is required. The other attributes allow you some manual control about which vault to connect to, but are only necessary if the application is not running in an environment that works magically with `Microsoft.Azure.Services.AppAuthentication`. The Azure Services Authentication library is used to automatically pick up connection information from the execution environment if possible, but you can override that feature by providing a connection string instead.

* `vaultName` - Required if `uri` in not provided. Specifies the name of the vault in your Azure subscription from which to read key/value pairs.
* `connectionString` - A connection string usable by [AzureServiceTokenProvider](https://docs.microsoft.com/en-us/azure/key-vault/service-to-service-authentication#connection-string-support)
* `uri` - Connects to other Key Vault providers with the specified `uri` value. If not specified, Azure (`vaultName`) is the vault provider.
* `version` - Azure Key Vault provides a versioning feature for secrets. If `version` is specified, the builder will only retrieve secrets matching this version.
* `preloadSecretNames` - By default, this builder will query **all** key names in the key vault when it is initialized. To prevent reading all key values, set this attribute to `false`. Setting this to `false` reads secrets one at a time. This could also be useful if the vault allows "Get" access but not
"List" access. **Note:** When using `Greedy` mode, `preloadSecretNames` must be `true` (The default.)

### KeyPerFileConfigBuilder

```xml
<add name="KeyPerFile"
    [mode|prefix|stripPrefix|tokenPattern]
	(directoryPath="PathToSourceDirectory")
    [ignorePrefix="ignore."]
    [keyDelimiter=":"]
    [optional="false"]
    type="Microsoft.Configuration.ConfigurationBuilders.KeyPerFileConfigBuilder,
    Microsoft.Configuration.ConfigurationBuilders.KeyPerFile" />
```

`KeyPerFileConfigBuilder` is a basic configuration builder that uses a directory's files as a source of values. A file's name is the key, and the contents are the value. This configuration builder can be useful when running in an orchestrated container environment. Systems like Docker Swarm and Kubernetes provide `secrets` to
their orchestrated windows containers in this key-per-file manner.

Attribute details:

* `directoryPath` - Required. Specifies a path to look in for values. Docker for Windows secrets are stored in the *C:\ProgramData\Docker\secrets* directory by default.
* `ignorePrefix` - Files that start with this prefix will be excluded. Defaults to "ignore.".
* `keyDelimiter` - Default value is `null`. If specified, the configuration builder will traverse multiple levels of the directory, building up key names with this delimiter. If this value is left `null`, the configuration builder only looks at the top level of the directory.
* `optional` -  Default value is `false`. Specifies whether the configuration builder should cause errors if the source directory doesn't exist.

### SimpleJsonConfigBuilder

```xml
<add name="SimpleJson"
    [mode|prefix|stripPrefix|tokenPattern]
    jsonFile="~\config.json"
    [optional="true"]
    [jsonMode="(Flat|Sectional)"]
    type="Microsoft.Configuration.ConfigurationBuilders.SimpleJsonConfigBuilder,
    Microsoft.Configuration.ConfigurationBuilders.Json" />
```

.Net Core projects frequently use JSON files for configuration. This builder allows .NET Core JSON files to be used in the .NET Framework. This configuration builder is meant provides a basic mapping from a flat key/value source into specific key/value areas of .NET Framework configuration. This configuration builder does **not** provide for hierarchical configurations. The JSON backing file is similar to a dictionary, not a complex hierarchical object. A multi-level hierarchical file can be used. This provider will `flatten` the depth by appending the property name at each level using `:` as a delimiter.

Attribute details:

* `jsonFile` - Required. Specifies the JSON file to draw from. The `~` character can be used at the start to reference the app root.
* `optional` - Boolean,  default value is `true`. Prevents throwing exceptions if the JSON file cannot be found.
* `jsonMode` - `[Flat|Sectional]`. `Flat` is the default. When `jsonMode` is `Flat`, the JSON file is a single flat key/value source. The `EnvironmentConfigBuilder` and `AzureKeyVaultConfigBuilder` are also single flat key/value sources. When the `SimpleJsonConfigBuilder` is configured in `Sectional` mode:

  * The JSON file is conceptually divided just at the top level into multiple dictionaries.
  * Each of the dictionaries will only be applied to the configuration section that matches the top-level property name attached to them. For example:

```json
    {
        "appSettings" : {
            "setting1" : "value1",
            "setting2" : "value2",
            "complex" : {
                "setting1" : "complex:value1",
                "setting2" : "complex:value2",
            }
        },

        "connectionStrings" : {
            "myJSON_ConnectionString" : "See security warning."
        }
    }
```

> [!WARNING]
> Never store passwords, sensitive connection strings, or other sensitive data in source code. Production secrets should not be used for development or test.

<!-- Review: I don't think barry AKA @blowdart will approve of a connection string here. I probably need this warning in other places. Please advise.  -->

## Implementing a custom key/value configuration builder

If the configuration builders don't meet your needs, you can write a custom one. The `KeyValueConfigBuilder` base class handles substitution modes and most prefix concerns. An implementing project needs only:

* Inherit from the base class and implement a basic source of key/value pairs through the `GetValue` and `GetAllValues`:
* Add the [Microsoft.Configuration.ConfigurationBuilders.Base](https://www.nuget.org/packages/Microsoft.Configuration.ConfigurationBuilders.Base/) to the project.

[!code-csharp[Main](config-builder/MyConfigBuilders/MyCustomConfigBuilder.cs)]

The `KeyValueConfigBuilder` base class provides much of the work and consistent behavior across key/value configuration builders.

The following sample implements a custom environment configuration builder:

[!code-csharp[Main](config-builder/MyConfigBuilders/EnvironmentConfigBuilder.cs)]

The following *web.config* markup enables the preceding custom environment configuration builder:

[!code-xml[Main](config-builder/MyConfigBuilders/WebCustom.config?name=snippet)]

## Additional resources

[Service-to-service authentication to Azure Key Vault using .NET](/azure/key-vault/service-to-service-authentication#connection-string-support)  