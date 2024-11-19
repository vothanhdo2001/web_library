
# Logger Code (C#)

## Installation
Before using the logger, ensure you have installed the **NLog** package. You can do this using NuGet Package Manager or the Package Manager Console:

### Using NuGet Package Manager Console
```bash
Install-Package NLog
```

### Using .NET CLI
```bash
dotnet add package NLog
```

## Namespace
```csharp
using NLog;
using NLog.Config;
using NLog.Targets;
using System;
using System.Configuration;
using System.IO;
```

## Class Definition
```csharp
public static class Log
{
    private static readonly Lazy<ILogger> lazyLogger = new Lazy<ILogger>(() =>
    {
        string logPath = ConfigurationManager.AppSettings["path"];
        if (string.IsNullOrEmpty(logPath))
        {
            throw new ConfigurationErrorsException("The 'path' app setting is missing in the configuration file.");
        }

        string timestamp = DateTime.Now.ToString("yyyy_MM_dd");
        string fileName = $"logfile_{timestamp}.log";
        string fullPath = Path.Combine(logPath, "Logs", fileName);

        var config = new LoggingConfiguration();
        var fileTarget = new FileTarget
        {
            FileName = fullPath,
            Layout = "${longdate} ${level:uppercase=true} ${message} ${exception}",

            ArchiveFileName = Path.Combine(logPath, "archives", "log.{#}.txt"),
            ArchiveEvery = FileArchivePeriod.Day,
            MaxArchiveFiles = 30
        };

        config.AddRule(LogLevel.Debug, LogLevel.Fatal, fileTarget);
        LogManager.Configuration = config;

        return LogManager.GetLogger(fileName);
    });

    private static ILogger Logger => lazyLogger.Value;

    public static void Info(string message)
    {
        try
        {
            Logger.Info(message);
            Console.ForegroundColor = ConsoleColor.White;
            Console.WriteLine($"INFO: {message}");
        }
        catch (Exception ex)
        {
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine($"Failed to log info: {ex.Message}");
        }
        finally
        {
            Console.ResetColor();
        }
    }

    public static void Error(string message, Exception ex = null)
    {
        try
        {
            Logger.Error(ex, message);
            Console.ForegroundColor = ConsoleColor.Red;
            Console.WriteLine($"ERROR: {message}");
            if (ex != null)
            {
                Console.WriteLine($"EXCEPTION: {ex.Message}");
            }
        }
        catch (Exception logEx)
        {
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine($"Failed to log error: {logEx.Message}");
        }
        finally
        {
            Console.ResetColor();
        }
    }

    public static void Warn(string message)
    {
        try
        {
            Logger.Warn(message);
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine($"WARN: {message}");
        }
        catch (Exception ex)
        {
            Console.ForegroundColor = ConsoleColor.Red;
            Console.WriteLine($"Failed to log warning: {ex.Message}");
        }
        finally
        {
            Console.ResetColor();
        }
    }

    public static void Initialize()
    {
        var _ = Logger;
        Log.Warn("===============START LOG================");
    }

    public static void FinalizeLogging()
    {
        Log.Warn("===============END LOG================");
    }
}
```
## Class Definition
```csharp
static void Main(string[] args)
{
    Log.Initialize();

    try
    {
        // Your application logic here
    }
    catch (Exception ex)
    {
        Log.Error("An unexpected error occurred", ex);
    }
    finally
    {
        Log.FinalizeLogging();
    }
}
```

