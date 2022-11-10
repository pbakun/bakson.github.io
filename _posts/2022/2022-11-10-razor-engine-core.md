---
layout: post
title: Templating with RazorEngineCore
tags: [docker, dotnet, playwright]
excerpt_separator: <!--more-->
---

Usually as .NET developers we can create neat HTML templates using powerful Razor Engine built into MVC, but rendering Razor template requires Controller's context. When we lack that Razor Engine Core comes in handy.
<!--more-->

This article is a follow up of [my previous post]({% post_url 2022-10-16-pdf-from-html %}), where I showed how to easily make PDF files out of HTML, but completely omitted how to pre-render it. There are many options here. I've seen people using [scriban](https://github.com/scriban/scriban), probably [liquid](https://github.com/mikebridge/Liquid.NET) would work as well, but most .NET developers are already familiar with Razor syntax. Normally it is always built into the framework and we don't have to think of how it is setup internally. Turns out using it is not that easy when doing it outside of ASP.NET MVC.

In case you're here to find out how to use built in the `System.Web.Mvc` Razor engine to render HTML without returning it as `View` this is the way:

```csharp
public string RazorViewToString<TModel>(ControllerContext controllerContext, string viewName, TModel model)
{
    ViewEngineResult viewEngineResult = ViewEngines.Engines.FindView(controllerContext, viewName, null);

    IView view = viewEngineResult.View;

    using (var stringWriter = new StringWriter())
    {
        var viewContext = new ViewContext(controllerContext, view, new ViewDataDictionary<TModel>(model), new TempDataDictionary(), stringWriter);
        view.Render(viewContext, stringWriter);

        return stringWriter.ToString();
    }
}
```
For those who, like I used to, struggle with using Razor Engine without `System.Web.Mvc` keep reading.

### The library

There is a github repo out there called [RazorEngineCore](https://github.com/adoconnection/RazorEngineCore). You can find there a lightweight, stripped to bones implementation of Razor engine. By default it only renders HTML string out of Razor formatted string, but with little work it can work pretty similar to known MVC's Razor. The downside is that most of stuff you need to implement yourself, but documentation provides how-to guides for most common cases.

### The setup

##### HTML safe & strongly typed template

To start with let's reuse [repository from previous post](https://github.com/pbakun/html-to-pdf/tree/pdf-from-html-post). Within the source folder I will create a new `Class Library` project - `HTML2PDF.RazorEngine`. First we need to add a reference to `RazorEngineCore`.

```powershell
Install-Package RazorEngineCore
```

Let's start with adding two classes: `HtmlSafeTemplate` and `RazorTemplate`. The first one is basically a copy paste from [documentation](https://github.com/adoconnection/RazorEngineCore/wiki/@Raw), with minor change. The second will only inherit from the first one. This will be the class later used for models within Razor templates.

```csharp
public class HtmlSafeTemplate<T> : RazorEngineTemplateBase<T>
{
    public object Raw(object value)
    {
        return new RawContent(value);
    }

    public override Task WriteAsync(object? obj = null)
    {
        object? value = obj is RawContent rawContent ? rawContent.Value : System.Web.HttpUtility.HtmlEncode(obj);
        return base.WriteAsync(value);
    }
    public override Task WriteAttributeValueAsync(string prefix, int prefixOffset, object? value, int valueOffset, int valueLength, bool isLiteral)
    {
        value = value is RawContent rawContent
            ? rawContent.Value
            : System.Web.HttpUtility.HtmlAttributeEncode(value?.ToString());

        return base.WriteAttributeValueAsync(prefix, prefixOffset, value, valueOffset, valueLength, isLiteral);
    }
}

public class RawContent
{
    public object Value { get; set; }
    public RawContent(object value)
    {
        Value = value;
    }
}
```

```csharp
public class RazorTemplate<T> : HtmlSafeTemplate<T>
{
}
```
In `HtmlSafeTemplate` class we override methods responsible for writing content from our properties and call `HTMLEncode` against them, so they become HTML safe. Additionally `Raw` method is added, which mimics behavior of MVC's `HTML.Raw`. The class inherits from `RazorEngineTemplateBase`. This is our base class, which is required by the lib. It is made generic, so it can be combined with any model. You could omit `<T>` and add properties for your model in `RazorTemplate` class.

To make our Razor Engine strongly typed we need to create a `CompiledTemplate` class with just one `Run` method and an extension method which will compile the template for us.

```csharp
public class CompiledTemplate<TTemplate, TModel> where TTemplate : RazorTemplate<TModel>
{
    private readonly IRazorEngineCompiledTemplate<TTemplate> _compiledTemplate;

    public CompiledTemplate(IRazorEngineCompiledTemplate<TTemplate> compiledTemplate)
    {
        _compiledTemplate = compiledTemplate;
    }

    public string Run(TModel model)
    {
        return _compiledTemplate.Run(instance =>
        {
            instance.Model = model;
        });
    }
}

public static class CompiledTemplateExtensions
{
    public static CompiledTemplate<RazorTemplate<TModel>, TModel>
        CompileTemplate<TModel>(this RazorEngineCore.RazorEngine razorEngine, string template)
    {
        return new CompiledTemplate<RazorTemplate<TModel>, TModel>(
            razorEngine.Compile<RazorTemplate<TModel>>(template, compilationOptionsBuilder));
    }

}
```

Basically we are forcing usage of `TModel`, which makes our Razor template strongly typed. Using this `CompiledTemplate` setup you could easily add possibility to include partial views or custom shared methods (which we'll do later). Before that let's add a `RazorService` which we will call to render HTML from Razor template.

```csharp
public interface IRazorService
{
    string CreateHtml<T>(T model, string templateView);
}

public class RazorService : IRazorService
{
    public string CreateHtml<T>(T model, string templateView)
    {
        RazorEngineCore.RazorEngine razorEngine = new RazorEngineCore.RazorEngine();
        CompiledTemplate<RazorTemplate<T>, T> compiled = razorEngine.CompileTemplate<T>(templateView);
        return compiled.Run(model);
    }
}
```

##### First test

Let's try to create an invoice. I have created a model and a `cshtml` template:

```csharp
public class Invoice
{
    public Contact Entrepreneur { get; set; }
    public Contact Recipient { get; set; }
    public string InvoiceNo { get; set; }
    public DateTime InvoiceDate { get; set; }
    public InvoiceEntry[] InvoiceEntries { get; set; }
}

public class Contact
{
    public string Fullname { get; set; }
    public string Address { get; set; }
    public string PostalCode { get; set; }
    public string City { get; set; }
}

public class InvoiceEntry
{
    public string Name { get; set; }
    public int Count { get; set; }
    public double Price { get; set; }
}
```
```html
@using HTML2PDF.Model.Template
@using HTML2PDF.RazorEngine.Templates

@inherits RazorTemplate<Invoice>

<!DOCTYPE html>
<html>
    <head>
	    <style>
			page {
                background: white;
                display: block;
                margin: 1cm auto;
                width: 21cm;
                height: 27.7cm;
                padding: 0 1cm !important;
                position: relative;
                page-break-after: always;
                font-family: Arial, Helvetica, sans-serif;
            }

            .heading {
                text-align: center;
            }

            .contact {
                margin: 25px 0;
            }

            .contact p {
                margin: 0;
                line-height: 1.3em;
            }
            .section-title {
                font-size: 16px;
                line-height: 1em;
                margin-bottom: 2px;
            }
            .section-title_translation {
                font-size: 12px;
            }

            .invoice-meta {
                display: flex;
                justify-content: space-between;
                margin-bottom: 5px;
            }

            table, th, td {
                border: 1px solid #000;
                border-collapse: collapse;
                width: 100%;
            }

            th:first-of-type ,td:first-of-type {
                min-width: 25px;
                width: min-content;
                text-align: left;
            }
            th:nth-of-type(2) ,td:nth-of-type(2) {
                width: 70%;
                text-align: left;
            }
            th:nth-of-type(3) ,td:nth-of-type(3) {
                width: 15%;
            }
            th, td {
                padding: 7px;
                text-align: center;
            }
		</style>
	</head>
	<body>
		<page>
            <h2 class="heading">Invoice</h2>
			<div class="contact">
                <div class="section-title">
                    Seller:
                </div>
				<p>@Model.Entrepreneur.Fullname</p>
				<p>@Raw(Model.Entrepreneur.Address)</p>
				<p>@Model.Entrepreneur.PostalCode @Model.Entrepreneur.City</p>
			</div>
            <div class="contact">
                <div class="section-title">
                    Recipient:
                </div>
				<p>@Model.Recipient.Fullname</p>
				<p>@Raw(Model.Recipient.Address)</p>
				<p>@Model.Recipient.PostalCode @Model.Recipient.City</p>
			</div>

            <div class="invoice-meta">
                <div class="invoice-no">
                    Invoice Nr: @Raw(Model.InvoiceNo)
                </div>
                <div class="invoice-date">@Raw(Model.InvoiceDate.Date.ToString("yyyy/MM/dd"))</div>
            </div>
            <table class="invoice-table">
                    <tr>
                        <th></th>
                        <th>Name</th>
                        <th>Count</th>
                        <th>Price (EUR)</th>
                    </tr>
                    @foreach(var (entry, index) in Model.InvoiceEntries.Select((value, i) => (value, i)))
                    {
                        <tr>
                            <td>@Raw(index+1).</td>
                            <td>@Raw(entry.Name)</td>
                            <td>@Raw(entry.Count)</td>
                            <td>@Raw(entry.Price)</td>
                        </tr>
                    }
            </table>
		</page>
	</body>
</html>
```

Unfortunately, it doesn't work with Playwright to attach linked assets from other files. You could set up additional web server which would serve those assets, but honestly I haven't tried it. If you don't have big, complex styling using `<style>` should be sufficient. Remember to mark `.cshtml` file as `None` and `Copy always` in file properties. As default Visual Studio creates all `.cshtml` files as `Content`, but in that case it will throw errors in compile as we don't have traditional Razor Engine installed in the project. This is how it should be included in `.csproj` file:

```xml
<ItemGroup>
    <None Update="Resources\img\invoice-logo.png">
        <CopyToOutputDirectory>Always</CopyToOutputDirectory>
    </None>
</ItemGroup>
```

The last necessary change for our test is to modify `PdfController` to get and compile Razor template.

```csharp
[Route("api/[controller]")]
[ApiController]
public class PdfController : ControllerBase
{
    private readonly IPdfService _pdfService;
    private readonly IRazorService _razorService;
    public const string FileDirectory = "test";

    public PdfController(IPdfService pdfService, IRazorService razorService)
    {
        _pdfService = pdfService;
        _razorService = razorService;
    }

    [HttpPost]
    public async Task<IActionResult> Create(Invoice invoice)
    {
        string path = Directory.GetCurrentDirectory() + "/Templates/Invoice.cshtml";
        var razorTemplate = await System.IO.File.ReadAllTextAsync(path);

        var html = _razorService.CreateHtml(invoice, razorTemplate);
        byte[] bytes = await _pdfService.CreateAsync(html);

        if (!Directory.Exists(FileDirectory))
        {
            Directory.CreateDirectory(FileDirectory);
        }
        await System.IO.File.WriteAllBytesAsync($"{FileDirectory}/hello.pdf", bytes);

        return Ok();
    }
}
```

As you can see I added `IRazorService` to controller's dependencies. In the `Create` POST method I read content of `Invoice.cshtml`, which is placed in `Templates` folder and call `CreateHtml` from `IRazorService` passing `cshtml` content and model taken from POST body. Of course in production code you would want to somehow cache templates content. You can try it yourself with `Invoice` sample body:

```json
{
    "entrepreneur": {
        "fullname": "Company A",
        "address": "Address 1/2",
        "postalCode": "1010",
        "city": "Vienna",
        "taxId": "string"
    },
    "recipient": {
        "fullname": "Company B",
        "address": "Address 2/1",
        "postalCode": "1010",
        "city": "Vienna",
        "taxId": "string"
    },
    "invoiceNo": "01/2022",
    "invoiceDate": "2022-11-08T21:34:22.449Z",
    "invoiceEntries": [
        {
        "name": "Service 1",
        "count": 1,
        "price": 1000
        },
        {
        "name": "Service 2",
        "count": 2,
        "price": 1000
        }
    ]
}
```

##### Template methods

I don't like how the invoice date is passed from the model, I'd like it to be just a date string in format `yyyy/MM/dd`. Let's try to do it through `DateTime` object, so we get an exception thrown in case of invalid format passed.

```html
<div class="invoice-meta">
    <div class="invoice-no">
        Invoice Nr: @Raw(Model.InvoiceNo)
    </div>
    <div class="invoice-date">@Raw(DateTime.ParseExact(Model.Invoice, "yyyy/MM/dd", CultureInfo.CurrentCulture).ToString("yyyy/MM/dd"))</div>
</div>
```

When we try to compile template with this piece of code in it will complain about not being aware of `DateTime` name. We can include additional references in the template, but it is advised to move as much logic as we can outside of it. If necessary we can always update our `RazorTemplate`:

```csharp
 public class RazorTemplate<T> : HtmlSafeTemplate<T>
{
    public string FormatDate(string dateString) => DateTime.ParseExact(dateString, "yyyy/MM/dd", CultureInfo.CurrentCulture).ToString("yyyy/MM/dd");
}
```

One more thing left. Every respected company needs a logo on their invoice. I quickly generated one using [logo.com](https://logo.com/). This is how it looks.

{% include center-img.html images="articles/2022/razor-engine-core/invoice-logo.png" %}

I have added logo section in my template just below header:
```html
<style>
    .company-logo {
        position: absolute;
        right: 50px;
        top: 30px;
    }
    .company-logo-img {
        max-width: 150px;
    }
</style>
<h2 class="heading">Invoice</h2>
<div class="company-logo">
    <img class="company-logo-img" src="./Resources/img/invoice-logo.png" />
</div>
```

After giving it a try turns out the image is not rendered. Why?

The reason is the same as the issue with linked assets I mentioned before. You could setup a web server which would serve those files or probably even add Controller in your ASP.NET Core app, but this time converting the image to base64 string should work as well. I will add one more method to `RazorTemplate` class and modify my `Invoice.cshtml` template.

```csharp
public string ConvertImageBase64String(string path)
{
    string base64Image = string.Empty;
    using (StreamReader reader = new StreamReader(path))
    {
        using (MemoryStream memoryStream = new MemoryStream())
        {
            reader.BaseStream.CopyTo(memoryStream);
            byte[] bytes = memoryStream.ToArray();
            string base64 = Convert.ToBase64String(bytes);
            base64Image = $"data:image/png;base64,{base64}";
        }
    }
    if (string.IsNullOrEmpty(base64Image))
        return string.Empty;
    return base64Image;
}
```

```html
<h2 class="heading">Invoice</h2>
<div class="company-logo">
    <img class="company-logo-img" src="@ConvertImageBase64String("./Resources/img/invoice-logo.png")" />
</div>
```

After second try I can review the new PDF file.

{% include center-img.html images="articles/2022/razor-engine-core/invoice-pdf.png" %}

### Wrap up

In this post I have showed how you can use [RazorEngineCore](https://github.com/adoconnection/RazorEngineCore) as template engine outside of ASP.NET MVC. In production environment you may want to add more stuff to it (eg. caching), but the purpose of this article was to show that it exists and is easy to use. I find it useful specially in combination with Playwright.NET. I&nbsp;encourage you to check out the project repository.

And [here](https://github.com/pbakun/html-to-pdf/tree/razor-engine-core) you can find repo with what I've done in this post. Till the next one!