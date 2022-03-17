# Surya.Ab.Logging
Current version of Surya.Ab.Logging internally uses [Serilog](https://github.com/serilog/serilog/wiki) to write Logs.

## Setup
Add “UseAbLog()” to the Generic Host in CreateHoldBuilder, and the pass the IConfiguration configuration.
```
public class Program
	{
		public static IConfiguration Configuration { get; set; }

		public static void Main(string[] args)
		{
			Configuration = new ConfigurationBuilder()
				.SetBasePath(Directory.GetCurrentDirectory())
				.AddJsonFile("appsettings.json")
				.AddJsonFile($"appsettings.{Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Development"}.json", true)
				.Build();

			CreateHostBuilder(args).Build().Run();
		}

		public static IHostBuilder CreateHostBuilder(string[] args) =>
			 Host.CreateDefaultBuilder(args)
				.UseAbLog(Configuration)
				  .ConfigureWebHostDefaults(webBuilder =>
				  {
					  webBuilder.UseStartup<Startup>();
				  });
	}
```

Initiallize the logger using Dependency Injection or one of the **GetCurrentClassLogger** methods.
```
private readonly ILogger _logger;

public MyClass(ILogger<MyClass> logger)
	{
		_logger = logger;
	}
```

## Configuration
Logger is configured using the 'appsettings.json' file.

- Configuration starts from the section named "Serilog".
- Set the “**MinimumLevel**’, only logs with level equal or greater will be logged
- “**Enrich**” section is used to add/remove additional properties:
	- “FromLogContext” ensures any properties added to the Log Context are pushed into log events
	- “WithClientIp” and “WithClientAgent” adds the ClientIp and ClientAgent respectively.
- “**WriteTo**” section specify the sink (output method) for Serilog to use:
	- In the configuration below, “WriteTo” is nested within another “WriteTo”, this is required when we want to have two sinks of the same type, which in this case is ‘File’.
	- For every logger we are **filtering** by log levels, “@l” specifies the log level
		“ByExcluding” can also be used in place of “ByIncludingOnly”

	- “**Name**” defines the type of sink 
	- “**path**” is a relative path to the log file (Note that if Rolling Interval is set, then date or the respective interval will be added before the extension)
	- “**rollingInterval**”: “Day” - setting rolling interval to day generates a new log file everyday.
	- "**fileSizeLimitBytes**" - specifies the size of log file in bytes (Serilog’s default is 1GB)
	- "**rollOnFileSizeLimit**" - if this is set to true, then a new file will be created once the fileSizeLimit is reached, if set to false, once the limit is reached log will not be logged
	- "**retainedFileCountLimit**": null - By default Seilog only retains the last 31 files, setting this to null makes sure that all log files are retained
	- "**formatter**": "Serilog.Formatting.Json.JsonFormatter, Serilog" - setting the formatter to JsonFormatter, changes the format of log messages to JSON.

> Ensure to wrap the file sink by 'Async' (improves performence by reducing the overhead of logging calls)

Sample Configuration:
```
"Serilog": {
		"MinimumLevel": {
			"Default": "Information"
		},
		"Enrich": [
			"FromLogContext",
			"WithClientIp",
			"WithClientAgent"
		],
		"WriteTo": [
			{
				"Name": "Async",
				"Args": {
					"configure": [
						{
							"Name": "Logger",
							"Args": {
								"configureLogger": {
									"Filter": [
										{
											"Name": "ByIncludingOnly",
											"Args": {
												"expression": "@l = 'Error' or @l = 'Fatal' "
											}
										}
									],
									"WriteTo": [
										{
											"Name": "File",
											"Args": {
												"path": "Logs/Error_Logs/Error_Logs_.txt",
												"rollingInterval": "Day",
												"fileSizeLimitBytes": 51200,
												"rollOnFileSizeLimit": true,
												"retainedFileCountLimit": null,
												"formatter": "Serilog.Formatting.Json.JsonFormatter, Serilog"
											}
										}
									]
								}
							}
						},
						{
							"Name": "Logger",
							"Args": {
								"configureLogger": {
									"Filter": [
										{
											"Name": "ByIncludingOnly",
											"Args": {
												"expression": "@l = 'Information' or @l = 'Warning'"
											}
										}
									],
									"WriteTo": [
										{
											"Name": "File",
											"Args": {
												"path": "Logs/Info_Logs/Info_Logs_.txt",
												"rollingInterval": "Day",
												"fileSizeLimitBytes": 51200,
												"rollOnFileSizeLimit": true,
												"retainedFileCountLimit": null,
												"formatter": "Serilog.Formatting.Json.JsonFormatter, Serilog"
											}
										}
									]
								}
							}
						},
						{
							"Name": "Logger",
							"Args": {
								"configureLogger": {
									"Filter": [
										{
											"Name": "ByIncludingOnly",
											"Args": {
												"expression": "@l = 'Debug'"
											}
										}
									],
									"WriteTo": [
										{
											"Name": "File",
											"Args": {
												"path": "Logs/Debug_Logs/Debug_Logs_.txt",
												"rollingInterval": "Day",
												"fileSizeLimitBytes": 51200,
												"rollOnFileSizeLimit": true,
												"retainedFileCountLimit": null,
												"formatter": "Serilog.Formatting.Json.JsonFormatter, Serilog"
											}
										}
									]
								}
							}
						}
					]
				}
			}
		]
	}
```
