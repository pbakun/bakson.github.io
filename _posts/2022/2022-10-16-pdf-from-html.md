---
layout: post
title: Create PDF from HTML Template using PLaywright.NET
tags: [docker, dotnet, playwright]
excerpt_separator: <!--more-->
---

Creating PDF in .NET is not that straight forward as one could imagine. There are many libraries, some of them paid, but with introduction of Playwright.NET a new way have shown up.
<!--more-->

[Playwright](https://playwright.dev/dotnet) was primarily designed to serve as a reliable end-to-end testing framework. It utilizes headless browser capabilities. I have many use cases for it, starting from testing, through automating reservation of doctor appointment, to creating PDF using HTML template. Playwright enables HTML injection and printing it like you would do that on your desktop's browser.

### Dig in!
Let us start with creating an empty ASP.NET Core Web API project and adding Playwright.NET package:
```
dotnet new webapi --name HTML2PDF
cd HTML2PDF
dotnet add package Microsoft.Playwright --version 1.27.0
```
Now create a `Service` folder within the project and add `PdfService` class:
```csharp
using Microsoft.Playwright;

public interface IPdfService
{
    Task<byte[]> CreateAsync(string html);
}

public class PdfService : IPdfService
{
    public async Task<byte[]> CreateAsync(string html)
    {
        using var playwright = await Playwright.CreateAsync();
        await using var browser = await playwright.Chromium.LaunchAsync(new BrowserTypeLaunchOptions { Headless = true });
        var page = await browser.NewPageAsync();
        await page.EmulateMediaAsync(new PageEmulateMediaOptions { Media = Media.Screen });
        await page.SetContentAsync(html, new PageSetContentOptions() { WaitUntil = WaitUntilState.Load });
        return await page.PdfAsync(new PagePdfOptions { Format = "A4" });
    }
}
```
That part of code is will be responsible for creating PDF. It initializes Playwright driver instance and opens browser in headless mode. Next it takes `html`, injects it into browser's page context and creates PDF out of it. The `PdfAsync` method has an options to pass path for produced file, but I prefer to return array of bytes and later decide what I will do with it. For the purpose of this post I will just save it in project root directory, but in real life I would prefer to send it into some kind of storage, eg. Azure Blob Storage.

Next add a controller with one HTTP POST method `Create` and run the application. Remember to also register our `PdfService` in `Program.cs` or your `Startup` class.
```csharp
builder.Services.AddSingleton<IPdfService, PdfService>();
```

```csharp
[Route("api/[controller]")]
[ApiController]
public class PdfController : ControllerBase
{
    private readonly IPdfService _pdfService;
    public const string FileDirectory = "test";

    public PdfController(IPdfService pdfService)
    {
        _pdfService = pdfService;
    }

    [HttpPost]
    public async Task<IActionResult> Create()
    {
        byte[] bytes = await _pdfService.CreateAsync("<h1>Hello World</h1>");

        if (!Directory.Exists(FileDirectory))
        {
            Directory.CreateDirectory(FileDirectory);
        }
        await System.IO.File.WriteAllBytesAsync($"{FileDirectory}/hello.pdf", bytes);

        return Ok();
    }
}
```

If you haven't installed Playwright before, or have a different version, instead of new PDF file you will probably get the following exception:
```
Microsoft.Playwright.PlaywrightException: Executable doesn't exist at C:\Users\Dell\AppData\Local\ms-playwright\chromium-1028\chrome-win\chrome.exe
╔════════════════════════════════════════════════════════════╗
║ Looks like Playwright was just installed or updated.       ║
║ Please run the following command to download new browsers: ║
║                                                            ║
║     pwsh bin/Debug/netX/playwright.ps1 install             ║
║                                                            ║
║ <3 Playwright Team                                         ║
╚════════════════════════════════════════════════════════════╝
```
The issue is you are missing browser(s) Playwright wants to use. To fix this you can follow instructions from exception or if you have `npm` installed you could use:
```powershell
npx playwright install
```
Now everything should be installed and application should be ready for tests. Run it, head to `Swagger` and send that `Create` HTTP POST request. If you look closely in project root directory you will notice new folder named `test`. It will contain `hello.pdf` file.

### Say no to dependencies

I could finish that post here, but I wouldn't be myself if I would not pack this app into docker image. As we have seen Playwright requires couple steps to set it up. I believe it is better to handle it once and later use it in any environment without the need to pre-install any dependencies. Just `docker run` and fun!

I always start with Visual Studio inbuilt Dockerfile. Right click on your project and then `Add -> Docker Support` to create ready-to-use template. We only have to change one line there - replace `base` ASP.NET runtime image with Playwright `focal` image (as of the day of writing only [focal](https://playwright.dev/dotnet/docs/docker) image has .NET SDK preinstalled). In this project I'm using .NET 6 and already checked that Playwright v1.27.0 image have the same version pre-installed. You can check how Playwright team builds their image [here](https://github.com/microsoft/playwright-dotnet/tree/main/utils/docker).

```bash
#FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base #instead of this
FROM mcr.microsoft.com/playwright/dotnet:v1.27.0-focal AS base #paste this
```

The second thing is to add one more property into `.csproj` file. You can add it in first `PropertyGroup` in the file:
```xml
<PlaywrightPlatform>Linux</PlaywrightPlatform>
```
Now try running the app in `Docker` mode and send that POST request again. It should go without errors and when you bash into the container you will find new pdf file in `/app/pdf` directory.

```bash
#To bash into container first find it name with
docker ps
#And then exec into it
docker exec -it <container-name-or-id> /bin/bash
```

### Wrap up
In this post I showed you how to easily create PDFs. HTML gives you enormous flexibility, you can use CSS or Javascript to modify how content looks like. At the end we packed our sample application into docker image, so you can use it anywhere with just one command.

In the next articles I am planning to show how HTML templates could be created with `Razor Engine` (and I don't mean `Razor Pages` you could know from `MVC`) and how document creation can be combined with messaging system of choice. Playwright basically launches new Chromium tab for each PDF so creating new documents with API requests might not be the best idea. Most of you probably joked about Chrome memory usage more than once.

You can find complete source code in the [repository](https://github.com/pbakun/html-to-pdf/tree/pdf-from-html-post). If you found that post useful please share it and if you have any question feel free to reach me out. Till the next one!