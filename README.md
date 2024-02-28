# GR8Tech.TestUtils.Logging

Author: Mykola Panasiuk

## TL;DR
This repository contains primary the source code for building NuGet packages to Open Source:
- GR8Tech.TestUtils.Logging
- GR8Tech.TestUtils.Logging.Extensions

## Goal
GR8Tech.TestUtils.Logging and GR8Tech.TestUtils.Logging.Extensions were developed to be a unified approach for logging in all test projects and test infrastructure NuGet packages.

Both packages and the approach for test logging is based on using **Serilog** under the hood. 

The main principles for logging are next:
- using one type of logger - **Serilog**
- write to Console with a standard and cross-team template
- have an option to flexibly on/off logs from different sources independently from each other
- be user-friendly and inject as low dependencies from other packages as possible 

The only one place where you have to use **GR8Tech.TestUtils.Logging** is your final project you are going to run (tests, console app, web app). Other NuGet packages may use simple Serilog interface without other restrictions. You may use **GR8Tech.TestUtils.Logging.Extensions** with some helpful extensions for Serilog.

## Settings file
Settings file might be named either `test-settings.json` or `appsettings.json`. It requires only to set log level for different sources.

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "System": "Warning",
        "Microsoft": "Warning",
        "My.First.Lib": "Verbose",
        "My.Second.Lib": "Debug"
      }
    }
  }
}
```
No need to provide your own Console template as template standard is integrated into the NuGet package.

## Important details
### Log Context
To achieve best results for logs you have to use class context in your logger, example:
```c#
namespace DebugLibFirst

public class First
{
     private ILogger _logger;

     public First()
     {
         _logger = SerilogDecorator.Logger
             .ForContext<First>()
     }
}
```
It will give you an option to change logs level from this source:
```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "DebugLibFirst": "Error"
      }
    }
  }
}
```

That works fine for non-static classes. If you want to have a consistent approach for logging, you have to add `SourceContext` for static classes as well. There are two options how to do that:

1) Fill logger context with SourceContext property:
    ```c#
    logger.ForContext(Constants.SourceContextPropertyName, typeof(MyStaticClass); 
    ```
2) Use extension method from NuGet `GR8Tech.TestUtils.Logging.Extensions`
    ```c#
    logger.ForContextStaticClass(typeof(MyStaticClass); 
    ```

### Log Level
Logs have different details on different log Levels, the less level, the more details you will see. Let's look at the `Information` log based on different log Levels.

`Original code`
```c#
using GR8Tech.TestUtils.Logging;
using GR8Tech.TestUtils.Logging.Extensions;

namespace MyNamespace
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var logger = SerilogDecorator.Logger
                .ForContext<Program>()
                .AddPrefix("mykola")
                .AddHiddenPrefix("panasiuk");

            var id = 11;
            var name = "great";
            var payload = new { id, name };

            logger
                .AddPayload("myPayload", payload)
                .Information("---> info");
        }
    }
}
```

`Level: Verbose`
```bash
[2023-11-13T14:17:02.5407790Z] [INF] [MyNamespace.Program] [mykola] [panasiuk] ---> info
 Properties:
   _myPayload = {"id":11,"name":"great"}
   env = localhost
   ThreadId = 1
   MachineName = MAC-77777
```

`Level: Debug`
```bash
[2023-11-13T14:17:25.6627680Z] [INF] [MyNamespace.Program] [mykola] [panasiuk] ---> info
 Payload:
   _myPayload = {"id":11,"name":"great"}
```

`Level: Information`
```bash
[2023-11-13T14:17:47.6331690Z] [INF] [mykola] ---> info
```

`Level: Warning or Error or Fatal`
```bash
no output
```

### Extensions from  GR8Tech.TestUtils.Logging.Extensions
You may use this NuGet Package to easily add `prefix`, `hidden_prefix`, `payload` and `SourceContext` of static class properties to log context.

Note: `payload` variable name should start with `_`, for example `_myVar` to be properly parsed in log template. Method `AddPayload()` handles that automatically 

```c#
using GR8Tech.TestUtils.Logging;
using GR8Tech.TestUtils.Logging.Extensions;

namespace MyNamespace
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var logger = SerilogDecorator.Logger
                .ForContext<Program>() // for static class use: .ForContextStaticClass(typeof(MyStaticClass))
                .AddPrefix("mykola")
                .AddHiddenPrefix("panasiuk");

            logger
                .AddPayload("myPayload", new object())
                .Information("---> info");
        }
    }
}
```

`Output for Verbose level`
```bash
[2023-11-13T14:24:00.8435030Z] [INF] [MyNamespace.Program] [mykola] [panasiuk] ---> info
 Properties:
   _myPayload = {"$type":"Object"}
   env = localhost
   ThreadId = 1
   MachineName = MAC-77777

```

Also there is an extension method `JsonFormattingIndented()` that makes it possible to easily switch payload serialization from single line to multiline formatting.


