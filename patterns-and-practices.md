# Patterns and practices

## Page settings

It is possible to put pages of various settings within the single document. Please notice that the example below declares two consecutive page sizes (A4 and A3) with various margin values:

```csharp{10-13,21-24}
public class StandardReport : IDocument
{
    // metadata

    public void Compose(IDocumentContainer container)
    {
        container
            .Page(page =>
            {
                page.MarginVertical(80);
                page.MarginHorizontal(100);
                page.PageColor(Colors.Grey.Medium); // transparent is default
                page.Size(PageSizes.A3);
                    
                page.Header().Element(ComposeHeader);
                page.Content().Element(ComposeBigContent);
                page.Footer().AlignCenter().PageNumber();
            })
            .Page(page =>
            {
                // you can specify multiple page types in the document
                // with independent configurations
                page.Margin(50)
                page.Size(PageSizes.A4);
                    
                page.Header().Element(ComposeHeader);
                page.Content().Element(ComposeSmallContent);
                page.Footer().AlignCenter().PageNumber();
            });
    }

    // content implementation
}
```

You easily change page orientation:
```csharp
// default is portrait
page.Size(PageSizes.A3);

// explicit portrait orientation
page.Size(PageSizes.A3.Portrait());

// change to landscape orientation
page.Size(PageSizes.A3.Landscape());
```

## Continuous page size

It is possible to define a page size with known width but dynamic height. In this example, the resulting page has constant width equal to A4 page's width, but its height depends on the content:

```csharp{13}
public class StandardReport : IDocument
{
    // metadata

    public void Compose(IDocumentContainer container)
    {
        container
            .Page(page =>
            {
                page.MarginVertical(40);
                page.MarginHorizontal(60);
                
                page.ContinuousSize(PageSizes.A4.Width);
                    
                page.Header().Element(ComposeHeader);
                page.Content().Element(ComposeContent);
                page.Footer().AlignCenter().PageNumber();
            });
    }

    // content implementation
}
```

::: danger
Because of the practical layout limitations, the maximum page height is limited to 14400 points (around 5 meters).
:::

## Length unit types

QuestPDF uses points as default measure unit. Where 1 inch equals to 72 points, according to PDF specification. However, the vast majority of the Fluent API supports additional/optional argument to specify unit type. 

| Unit       | Size                 |
|------------|----------------------|
| meter      | 100 centimeters      |
| centimetre | 2.54 inches          |
| millimetre | 1/10th of centimeter |
| feet       | 12 inches            |
| inch       | 72 points            |
| mill       | 1/1000th of inch     |

Example usage:

```csharp
// all invocations are equal
.Padding(72)
.Padding(1, Unit.Inch)
.Padding(1/12f, Unit.Feet)
.Padding(1000, Unit.Mill)

// unit types can be provided in other API methods too, e.g.
.BorderLeft(100, Unit.Mill)
row.ConstantItem(8, Unit.Centimetre)
```


## Execution order

QuestPDF uses FluentAPI and method chaining to describe document's content. It is very important to remember that order of methods os strict. That means, in many cases, changing order of invocations will produce different results. To better understand this behavior, let's analyse this simple example:

```csharp{7-8,13-14}
.Row(row =>
{
    row.Spacing(25);

    row.RelativeItem()
        .Border(1)
        .Padding(15)
        .Background(Colors.Grey.Lighten2)
        .Text("Lorem ipsum");
    
    row.RelativeItem()
        .Border(1)
        .Background(Colors.Grey.Lighten2)
        .Padding(15)
        .Text("dolor sit amet");
});
```

![example](./images/patterns-and-practices/execution-order-1.png =400x)

This is another good example showing how applying padding changes available space:

```csharp
.Padding(25)
.Border(2)
.Width(150)
.Height(150)

.Background(Colors.Blue.Lighten2)
.PaddingTop(50)

.Background(Colors.Green.Lighten2)
.PaddingRight(50)

.Background(Colors.Red.Lighten2)
.PaddingBottom(50)

.Background(Colors.Amber.Lighten2)
.PaddingLeft(50)

.Background(Colors.Grey.Lighten2);
```

![example](./images/patterns-and-practices/execution-order-2.png =200x)

## Global text style

The QuestPDF library provides a default set of styles that applied to text.

```csharp
.Text("Text with library default styles")
```

You can adjust the text style by providing additional argument:

```csharp
.Text("Red semibold text of size 20").FontSize(20).SemiBold()
```

The option above introduces overrides the default style. To get more control you can set a default text style in your document. Please notice that all changes are additive:

```csharp{9-10,22-23,27-28}
public class SampleReport : IDocument
{
    public DocumentMetadata GetMetadata() => new DocumentMetadata();

    public void Compose(IDocumentContainer container)
    {
        container.Page(page =>
        {
            // all text in this set of pages has size 20
            page.DefaultTextStyle(TextStyle.Default.FontSize(20));
            
            page.Margin(20);
            page.Size(PageSizes.A4);
            page.PageColor(Colors.White);

            page.Content().Column(column =>
            {
                column.Item().Text(Placeholders.Sentence());
                
                column.Item().Text(text =>
                {
                    // text in this block is additionally semibold
                    text.DefaultTextStyle(x => x.SemiBold());

                    text.Line(Placeholders.Sentence());
                    
                    // this text has size 20 but also semibold and red
                    text.Span(Placeholders.Sentence()).FontColor(Colors.Red.Medium);
                });
            });
        });
    }
}
```

![example](./images/patterns-and-practices/global-text-style.png =595x)

## Document metadata

You can modify the PDF document metadata by returning the `DocumentMetadata` object from the `IDocument.GetMetadata()` method. There are multiple properties available, some of them have default values:

```csharp
public class DocumentMetadata
{
    public int ImageQuality { get; set; } = 101;
    public int RasterDpi { get; set; } = 72;
    public bool PdfA { get; set; }

    public string? Title { get; set; }
    public string? Author { get; set; }
    public string? Subject { get; set; }
    public string? Keywords { get; set; }
    public string? Creator { get; set; }
    public string? Producer { get; set; }

    public DateTime CreationDate { get; set; } = DateTime.Now;
    public DateTime ModifiedDate { get; set; } = DateTime.Now;

    public int DocumentLayoutExceptionThreshold { get; set; } = 250;

    public bool ApplyCaching { get; set; } // false when debugger is attached
    public bool ApplyDebugging { get; set; } // true when debugger is attached

    public static DocumentMetadata Default => new DocumentMetadata();
}
```

::: tip
If the number of generated pages exceeds the `DocumentLayoutExceptionThreshold` (likely due to infinite layout), the exception is thrown. Please adjust this parameter, so the library can stop as soon as possible, saving CPU and memory resources.
:::

::: tip
The `ImageQuality` property controls the trade-off between quality and size. The default value `101` corresponds to lossless encoding. When you use a value less than 100, all images are opaque and encoded using the JPEG algorithm. The smaller the value is, the higher compression is used.
:::

## Generating PDF and XPS files

The library supports generating both PDF and XPS files:

```csharp
report.GeneratePdf("result.pdf");
report.GenerateXps("result.xps");
```


## Generating images

The default functionality of the library is generating PDF files based on specified document configuration. In some cases, you may need to generate set of images instead. Such tasks can be done by additional extension methods:

```csharp
// generate images as dynamic list of images
IEnumerable<byte[]> images = document.GenerateImages();

// generate images and save them as files with provided naming schema
document.GenerateImages(i => $"image-{i}.png");  
```

::: tip
Generated images are in the PNG format. In order to increase resolution of generated images, please modify the value of the `DocumentMetadata.RasterDpi` property. When RasterDpi is set to 72, one PDF point corresponds to one pixel.
:::

## Support for custom environments (cloud / linux)

The QuestPDF library has a dependency called SkiaSharp which is used to render the final PDF file. This library has additional dependencies when used in the Linux environment. 

When you get the exception `Unable to load shared library 'libSkiaSharp' or one of its dependencies.`, please try to install additional nuget packages provided by the SkiaSharp team.

For example: the SkiaSharp.NativeAssets.Linux.NoDependencies nuget ensures that the libSkiaSharp.so file is published and available.

## Support for custom environments (Blazor WebAssembly)

The QuestPDF library has a dependency called SkiaSharp which is used to render the final PDF file. 
QuestPDF works without problems in Blazor WebAssembly but you need to provide a suitable version of SkiaSharp on runtime. 

**Note:** The tools used in this section are in a prerelease state, they should be used with caution.

First you need to install the WebAssembly tools, that will be used to compile the Skia dependencies to WebAssembly. In a command shell, execute

```
dotnet workload install wasm-tools
```

Then you will need to add the SkiaSharp.Views.Blazor Nuget package to your project. SkiaSharp is a crossplatform graphics library based on Skia.

```
dotnet add package –-prerelease SkiaSharp.Views.Blazor
```

Subsecuent builds can be slower as the toolchain now compiles Skia to WebAssembly.

Another thing to consider is that WebAssembly code runs on a secured sandbox and it can't access system fonts.
For this reason any PDF created with QuestPDF in Blazor WebAssembly will not embed the used fonts. The final font used for rendering will be determined by the PDF viewer.
However you can use a custom font as shown in the section **Accessing custom fonts**.


## Accessing custom fonts

The QuestPDF library has access to all fonts installed on the hosting system. Sometimes though, you don't have control over fonts installed on the production environment. Or you may want to use self-hosted fonts that come with your application as files or embedded resources. In such case, you need to register those fonts as follows:

```csharp{2,5,13}
// static method definition
FontManager.RegisterFont(Stream fontDataStream);

// perform similar invocation only once, when the application starts or during its initialization step
FontManager.RegisterFont(File.OpenRead("LibreBarcode39-Regular.ttf")); // use file name

// then, you will have access to the font by its name
container
    .Background(Colors.White)
    .AlignCenter()
    .AlignMiddle()
    .Text("*QuestPDF*")
    .FontFamily("Libre Barcode 39") // use real font family name
    .FontSize(64);
```

This way, it is possible to generate barcodes:

![example](./images/patterns-and-practices/custom-font.png =400x)

## Extending DSL

The existing Fluent API offers a clear and easy-to-understand way to describe a document's structure. When working on the document, you may find that many places use similar styles, for instances borders or backgrounds. It is especially common when you keep the document consistent. To make future adjustments easier, you can reuse the styling code by extracting it into separate extension methods. This way, you can assign a meaningful name to documents structure without increasing code complexity.

In the example below, we will create a simple table where label cells have a grey background, and value cells have a white background. First, let's create proper extension methods:

```csharp
static class SimpleExtension
{
    private static IContainer Cell(this IContainer container, bool dark)
    {
        return container
            .Border(1)
            .Background(dark ? Colors.Grey.Lighten2 : Colors.White)
            .Padding(10);
    }
    
    // displays only text label
    public static void LabelCell(this IContainer container, string text) => container.Cell(true).Text(text).Medium();
    
    // allows to inject any type of content, e.g. image
    public static IContainer ValueCell(this IContainer container) => container.Cell(false);
}
```

Now, you can easily use newly created DSL language to build the table:

```csharp
.Grid(grid =>
{
    grid.Columns(10);
    
    for(var i=1; i<=4; i++)
    {
        grid.Item(2).LabelCell(Placeholders.Label());
        grid.Item(3).ValueCell().Image(Placeholders.Image(200, 150));
    }
});
```

This example produces the following output:

![example](./images/patterns-and-practices/domain-specific-language.png =600x)

::: tip
Please note that this example shows only the concept of using extension methods to build custom API elements. Using this approach you can build and reuse more complex structures. For example, extension methods can expect arguments.
:::

## Complex layouts and grids

By combining various elements, you can build complex layouts. When designing try to break your layout into separate pieces and then model them by using the `Row` and the `Column` elements. In many cases, the `Grid` element can simplify and shorten your code.

Please consider the code below. Please note that it uses example DSL elements from the previous section.

```csharp
.Column(column =>
{
    column.Item().Row(row =>
    {
        row.RelativeItem().LabelCell("Label 1");
        
        row.RelativeItem(3).Grid(grid =>
        {
            grid.Columns(3);
            
            grid.Item(2).LabelCell("Label 2");
            grid.Item().LabelCell("Label 3");
            
            grid.Item(2).ValueCell().Text("Value 2");
            grid.Item().ValueCell().Text("Value 3");
        });
    });
    
    column.Item().Row(row =>
    {
        row.RelativeItem().ValueCell().Text("Value 1");
        
        row.RelativeItem(3).Grid(grid =>
        {
            grid.Columns(3);
            
            grid.Item().LabelCell("Label 4");
            grid.Item(2).LabelCell("Label 5");
            
            grid.Item().ValueCell().Text("Value 4");
            grid.Item(2).ValueCell().Text("Value 5");
        });
    });
    
    column.Item().Row(row =>
    {
        row.RelativeItem().LabelCell("Label 6");
        row.RelativeItem().ValueCell().Text("Value 6");
    });
});
```

And its corresponding output:

![example](./images/patterns-and-practices/complex-layout.png =500x)

::: tip
When implementing complex grids and layouts, please also consider using the `Table` element. [Click here to learn more](/api-reference.html#table)
:::

## Ordered and bullet lists

Implementing lists is very simple with the `Column` and `Row` elements:

```csharp
.Column(column =>
{
    foreach (var i in Enumerable.Range(1, 8))
    {
        column.Item().Row(row =>
        {
            row.Spacing(5);
            row.AutoItem().Text($"{i}."); // text or image
            row.RelativeItem().Text(Placeholders.Sentence());
        });
    }
});
```

The result is as follows:

![example](./images/patterns-and-practices/ordered-list.png =500x)

## Components

A component is a special type of element that can generate content depending on its state. This approach is really common in many web-development libraries and solves multiple issues. You should consider creating your own component when part of the document is going to be reused in other documents. Another good scenario is when you plan to repeat a more complex section. In such a case, you can implement a component that takes input provided as constructor's argument, and generates PDF content. Then, such component can be easily used in a for loop in the document itself. All things considered, components are a useful tool to organize and reuse your code.

::: tip
Components offer a lot of flexibility and extendability. Because of that, the QuestPDF library will receive several important updates to enhance components features even more. Stay tuned for slots!
:::

In this tutorial, we will cover a simple component that generates a random image taken from the fantastic webpage called [Lorem Picsum](https://picsum.photos/). To show how component's behaviour can be dynamically changed, the end result will offer optional greyscale flag.
 
 Additionally, the constructor of the template is going to offer of showing only greyscale images.

```csharp
//interface
public interface IComponent
{
    void Compose(IContainer container);
}

// example implementation
public class LoremPicsum : IComponent
{
    public bool Greyscale { get; }

    public LoremPicsum(bool greyscale)
    {
        Greyscale = greyscale;
    }
    
    public void Compose(IContainer container)
    {
        var url = "https://picsum.photos/300/200";

        if(Greyscale)
            url += "?grayscale";

        using var client = new WebClient();
        var response = client.DownloadData(url);
        container.Image(response);
    }
}
```

Example usage:

```csharp{7}
.Column(column =>
{
    column.Spacing(10);

    column
        .Element()
        .Component(new LoremPicsum(true));
    
    column
        .Element()
        .AlignRight()
        .Text("From Lorem Picsum");
});
```

The result of sample code looks as follows:

![example](./images/patterns-and-practices/component-example.png =350x)

::: tip
If the component class has parameter-less constructor, you can use the generic `Template` method like so:
```csharp
.Component<ComponentClass>();
```
:::


## Dynamic components

### Introduction

Dynamic components are useful when you want to generate different or conditional content on each page. The dynamic component mostly resembles normal components with one important difference: the `Compose` method is called for each page. Having access to component internal state, information about pages and available space, you can build more advanced structures.

In this example, we only use the page information: number of current page and count of all pages in the document. We use both numbers to create something similar to a progress bar that shows where you are in the document.

Please note that this example does not require state management. Therefore, we declare state as a simple integer and do not use it anywhere else.

```csharp{7-22}
public class ProgressHeader : IDynamicComponent<int>
{
    public int State { get; set; }
    
    public DynamicComponentComposeResult Compose(DynamicContext context)
    {
        var content = context.CreateElement(container =>
        {
            var width = context.AvailableSize.Width * context.PageNumber / context.TotalPages;
            
            container
                .Background(Colors.Blue.Lighten2)
                .Height(25)
                .Width(width)
                .Background(Colors.Blue.Darken1);
        });

        return new DynamicComponentComposeResult
        {
            Content = content,
            HasMoreContent = false
        };
    }
}
```

Using the dynamic component is very simple. Just use the `Dynamic` method and provide an instance of your component.

```csharp{7}
container.Page(page =>
{
    page.Size(PageSizes.A6);
    page.Margin(1, Unit.Centimetre);
    page.DefaultTextStyle(x => x.FontSize(20));

    page.Header().Dynamic(new ProgressHeader());
    
    page.Content().Column(column =>
    {
        foreach (var i in Enumerable.Range(0, 100))
            column.Item().PaddingTop(25).Background(Colors.Grey.Lighten2).Height(50);
    });

    page.Footer().AlignCenter().Text(text =>
    {
        text.CurrentPageNumber();
        text.Span(" / ");
        text.TotalPages();
    });
});
```

First page of the document, and one random page from the middle of the document:

![example](./images/patterns-and-practices/dynamic-progress-1.png =300x)
![example](./images/patterns-and-practices/dynamic-progress-2.png =300x)

### Footer with alternating text alignment

In this example, we will use new knowledge to implement the footer element with alternating text alignment.
1) On even pages, align page number to the left.
2) On odd pages, align page number to the right.

For example:

```
Page 1 -> Text aligned right.
Page 2 -> Text aligned left.
Page 3 -> Text aligned right.
...
and so on...
```

The code is quite simple:

```csharp{10}
public class FooterWithAlternatingAlignment : IDynamicComponent<int>
{
    public int State { get; set; }
    
    public DynamicComponentComposeResult Compose(DynamicContext context)
    {
        var content = context.CreateElement(element =>
        {
            element
                .Element(x => context.PageNumber % 2 == 0 ? x.AlignLeft() : x.AlignRight())
                .Text(x =>
                {
                    x.CurrentPageNumber();
                    x.Span(" / ");
                    x.TotalPages();
                });
        });
        
        return new DynamicComponentComposeResult()
        {
            Content = content,
            HasMoreContent = false
        };
    }
}
```

And the result is as follows:

![example](./images/patterns-and-practices/dynamic-alternating-footer-1.png =300x)
![example](./images/patterns-and-practices/dynamic-alternating-footer-2.png =300x)

### State management

This example introduces state management in components. Our goal is to create a header component that calculates consecutive terms in the Fibonacci-like sequence and shows their ratio - one calculation per page. Additionally, we want to use different background colors depending on the modulo calculus of the current sequence term.

Let's begin with the struct declaration that will hold the state:

::: warning
Important: please consider the state as read only. Never mutate already existing state. To perform mutation, create new struct instance and assign it to the State property. The QuestPDF library may perform multiple `Compose` method calls per page. The library may also change the state internally.
:::

```csharp
public struct FibonacciHeaderState
{
    public int Previous { get; set; }
    public int Current { get; set; }
}
```

Now, in each `Compose` method invocation, we can calculate new sequence term, properly update state and generate new content to display on the page. 

```csharp{23-49}
public class FibonacciHeader : IDynamicComponent<FibonacciHeaderState>
{
    public FibonacciHeaderState State { get; set; }
    
    public static readonly string[] ColorsTable =
    {
        Colors.Red.Lighten2,
        Colors.Orange.Lighten2,
        Colors.Green.Lighten2,
    };

    public FibonacciHeader(int previous, int current)
    {
        State = new FibonacciHeaderState
        {
            Previous = previous,
            Current = current
        };
    }

    public DynamicComponentComposeResult Compose(DynamicContext context)
    {
        var content = context.CreateElement(container =>
        {
            var colorIndex = State.Current % ColorsTable.Length;
            var color = ColorsTable[colorIndex];

            var ratio = (float)State.Current / State.Previous;
            
            container
                .Background(color)
                .Height(50)
                .AlignMiddle()
                .AlignCenter()
                .Text($"{State.Current} / {State.Previous} = {ratio:N5}");
        });

        // please notice that the code assign NEW state, instead of mutating existing one
        State = new FibonacciHeaderState
        {
            Previous = State.Current,
            Current = State.Previous + State.Current
        };
        
        return new DynamicComponentComposeResult
        {
            Content = content,
            HasMoreContent = false // each page has its own content
        };
    }
}
```

Please notice that you can instantiate components using constructors with arguments. This way for example, you can pass data from the database:

```csharp{7}
page.Header().Dynamic(new FibonacciHeader(17, 19));
```

First page of the document, and one random page from the middle of the document:

![example](./images/patterns-and-practices/dynamic-state-1.png =300x)
![example](./images/patterns-and-practices/dynamic-state-2.png =300x)

### Table with per-page totals

This example presents more common use case. Similarly to the Getting Started tutorial, we will generate an invoice document. The difficulty is hidden in the special requirement: for each page on the invoice, we want to show the total price only for items visible on the page. Therefore, we need to know which items are visible on the page and then calculate the proper value.

To achieve this requirement, we will implement a simple paging algorithm that will check how many table rows can fit on the page. 

Let's begin with a declaring data model and state struct:

```csharp
public class OrderItem
{
    public string ItemName { get; set; } = Placeholders.Label();
    public int Price { get; set; } = Placeholders.Random.Next(1, 11) * 10;
    public int Count { get; set; } = Placeholders.Random.Next(1, 11);
}

public struct OrdersTableState
{
    public int ShownItemsCount { get; set; }
}
```

The implementation of this component is quite simple. For each page, the component generates multiple versions of the layout, testing how much space is required to various number of items in the table. Please notice that the `DynamicContent.CreateElement` method returns an object implementing the `IDynamicElement` interface. This interface can be used to access size of the element. This size can be compared to available space, so the biggest table with the highest number of rows is chosen.

```csharp
public class OrdersTable : IDynamicComponent<OrdersTableState>
{
    private IList<OrderItem> Items { get; }
    public OrdersTableState State { get; set; }

    public OrdersTable(IList<OrderItem> items)
    {
        Items = items;

        State = new OrdersTableState
        {
            ShownItemsCount = 0
        };
    }
    
    public DynamicComponentComposeResult Compose(DynamicContext context)
    {
        // try to generate multiple layout versions (tables with various number of rows)
        // and pick the biggest table that still fits in the available space
        var possibleItems = Enumerable
            .Range(1, Items.Count - State.ShownItemsCount)
            .Select(itemsToDisplay => ComposeContent(context, itemsToDisplay))
            .TakeWhile(x => x.Size.Height <= context.AvailableSize.Height)
            .ToList();

        // update the state, so the component remembers how many items has been already shown on previous pages
        State = new OrdersTableState
        {
            ShownItemsCount = State.ShownItemsCount + possibleItems.Count
        };

        return new DynamicComponentComposeResult
        {
            Content = possibleItems.Last(),
            
            // check, if all items has been already rendered
            HasMoreContent = State.ShownItemsCount < Items.Count
        };
    }

    private IDynamicElement ComposeContent(DynamicContext context, int itemsToDisplay)
    {
        // this method is called multiple times per page
        // with each calls, the value of the 'itemsToDisplay' argument increases
    
        var total = Items.Skip(State.ShownItemsCount).Take(itemsToDisplay).Sum(x => x.Count * x.Price);

        return context.CreateElement(container =>
        {
            container
                .MinimalBox()
                .Width(context.AvailableSize.Width) // please notice that we need to constraing the element's width to available space
                .Table(table =>
                {
                    table.ColumnsDefinition(columns =>
                    {
                        columns.ConstantColumn(30);
                        columns.RelativeColumn();
                        columns.ConstantColumn(50);
                        columns.ConstantColumn(50);
                        columns.ConstantColumn(50);
                    });
                    
                    table.Header(header =>
                    {
                        header.Cell().Element(Style).Text("#");
                        header.Cell().Element(Style).Text("Item name");
                        header.Cell().Element(Style).AlignRight().Text("Count");
                        header.Cell().Element(Style).AlignRight().Text("Price");
                        header.Cell().Element(Style).AlignRight().Text("Total");

                        IContainer Style(IContainer container)
                        {
                            return container
                                .DefaultTextStyle(x => x.SemiBold())
                                .BorderBottom(1)
                                .BorderColor(Colors.Grey.Darken2)
                                .Padding(5);
                        }
                    });
                    
                    table.Footer(footer =>
                    {
                        footer
                            .Cell().ColumnSpan(5)
                            .AlignRight()
                            .Text($"Subtotal: {total}$", TextStyle.Default.Size(14).SemiBold());
                    });
                    
                    foreach (var index in Enumerable.Range(State.ShownItemsCount, itemsToDisplay))
                    {
                        var item = Items[index];
                            
                        table.Cell().Element(Style).Text(index + 1);
                        table.Cell().Element(Style).Text(item.ItemName);
                        table.Cell().Element(Style).AlignRight().Text(item.Count);
                        table.Cell().Element(Style).AlignRight().Text($"{item.Price}$");
                        table.Cell().Element(Style).AlignRight().Text($"{item.Count*item.Price}$");

                        IContainer Style(IContainer container)
                        {
                            return container
                                .BorderBottom(1)
                                .BorderColor(Colors.Grey.Lighten2)
                                .Padding(5);
                        }
                    }
                });
        });
    }
}
```

```csharp
// generate random data
var items = Enumerable.Range(0, 25).Select(x => new OrderItem()).ToList();

container
    .Background(Colors.White)
    .Padding(25)
    .Decoration(decoration =>
    {
        decoration
            .Header()
            .PaddingBottom(5)
            .Text(text =>
            {
                text.DefaultTextStyle(TextStyle.Default.SemiBold().FontColor(Colors.Blue.Darken2).FontSize(16));
                text.CurrentPageNumber();
                text.Span(" / ");
                text.TotalPages();
            });
        
        decoration
            .Content()
            .Dynamic(new OrdersTable(items));
    });
```

![example](./images/patterns-and-practices/dynamic-subtotals-1.png =300x)
![example](./images/patterns-and-practices/dynamic-subtotals-2.png =300x)

### Optimized example

In the previous example, we generate and measure multiple different sizes (with different number of rows) of table for each page. Therefore, to render one page, we may generate over 10-15 different layout versions, thus making the algorithm not optimal.

This example uses the `IDynamicElement.Size` information to better manage generation process:
1) Generate table header and measure how much space it needs
2) Generate table footer (subtotal) and measure required space.
3) Incrementally generate table rows, measure each row, and fill available space. Continue till the next row does not fit and reject it.

This way, we can build the table only once, significantly improving performance. Of course, this algorithm is slightly more complicated:

```csharp
public class OptimizedOrdersTable : IDynamicComponent<OrdersTableState>
{
    private ICollection<OrderItem> Items { get; }
    public OrdersTableState State { get; set; }

    public OptimizedOrdersTable(ICollection<OrderItem> items)
    {
        Items = items;

        State = new OrdersTableState
        {
            ShownItemsCount = 0
        };
    }
    
    public DynamicComponentComposeResult Compose(DynamicContext context)
    {
        var header = ComposeHeader(context);
        var sampleFooter = ComposeFooter(context, Enumerable.Empty<OrderItem>());
        var decorationHeight = header.Size.Height + sampleFooter.Size.Height;
        
        var rows = GetItemsForPage(context, decorationHeight).ToList();
        var footer = ComposeFooter(context, rows.Select(x => x.Item));

        var content = context.CreateElement(container =>
        {
            container.MinimalBox().Decoration(decoration =>
            {
                decoration.Header().Element(header);

                decoration.Content().Box().Stack(stack =>
                {
                    foreach (var row in rows)
                        stack.Item().Element(row.Element);
                });

                decoration.Footer().Element(footer);
            });
        });

        State = new OrdersTableState
        {
            ShownItemsCount = State.ShownItemsCount + rows.Count
        };

        return new DynamicComponentComposeResult
        {
            Content = content,
            HasMoreContent = State.ShownItemsCount < Items.Count
        };
    }

    private IDynamicElement ComposeHeader(DynamicContext context)
    {
        return context.CreateElement(element =>
        {
            element
                .Width(context.AvailableSize.Width)
                .BorderBottom(1)
                .BorderColor(Colors.Grey.Darken2)
                .Padding(5)
                .Row(row =>
                {
                    var textStyle = TextStyle.Default.SemiBold();

                    row.ConstantItem(30).Text("#", textStyle);
                    row.RelativeItem().Text("Item name", textStyle);
                    row.ConstantItem(50).AlignRight().Text("Count", textStyle);
                    row.ConstantItem(50).AlignRight().Text("Price", textStyle);
                    row.ConstantItem(50).AlignRight().Text("Total", textStyle);
                });
        });
    }
    
    private IDynamicElement ComposeFooter(DynamicContext context, IEnumerable<OrderItem> items)
    {
        var total = items.Sum(x => x.Count * x.Price);

        return context.CreateElement(element =>
        {
            element
                .Width(context.AvailableSize.Width)
                .Padding(5)
                .AlignRight()
                .Text($"Subtotal: {total}$", TextStyle.Default.Size(14).SemiBold());
        });
    }
    
    private IEnumerable<(OrderItem Item, IDynamicElement Element)> GetItemsForPage(DynamicContext context, float decorationHeight)
    {
        var totalHeight = decorationHeight;

        foreach (var index in Enumerable.Range(State.ShownItemsCount, Items.Count - State.ShownItemsCount))
        {
            var item = Items.ElementAt(index);
            
            var element = context.CreateElement(content =>
            {
                content
                    .Width(context.AvailableSize.Width)
                    .BorderBottom(1)
                    .BorderColor(Colors.Grey.Lighten2)
                    .Padding(5)
                    .Row(row =>
                    {
                        row.ConstantItem(30).Text(index + 1);
                        row.RelativeItem().Text(item.ItemName);
                        row.ConstantItem(50).AlignRight().Text(item.Count);
                        row.ConstantItem(50).AlignRight().Text($"{item.Price}$");
                        row.ConstantItem(50).AlignRight().Text($"{item.Count*item.Price}$");
                    });
            });

            var elementHeight = element.Size.Height;
                
            if (totalHeight + elementHeight > context.AvailableSize.Height)
                break;
                
            totalHeight += elementHeight;
            yield return (item, element);
        }
    }
}
```

## Reducing output size

### Use loss compression for images

By default, QuestPDF uses a lossless compression algorithm. If your document contains a lot of images, the output file size can become bigger than expected. Please consider using the loss compression. It is done by changing the `DocumentMetadata.ImageQuality` property. The precise instruction how this property works can be found in [this section](http://localhost:8080/documentation/patterns-and-practices.html#document-metadata).

### Perform manual font subsetting

QuestPDF is a layout engine that uses the SkiaSharp library to render final PDF file. Therefore, it derives some limitations. One of them is how fonts are handled. By default, SkiaSharp embeds all fonts within the PDF document. Each font may consist of thousands of glyphs, with style and weight variations. Therefore, using even a single font may increase the output size to over 1,5MB.

When designing the document, please try to limit number of fonts used.

Consider performing a manual font sub-setting. This is a process where you alter the font file by removing all unnecessary glyphs. For example, if your document uses only english characters, there is no reason to embed Chinese glyphs. They are just taking space. By predicting which characters will appear in your document, you can easily optimize fonts.

::: tip
Current discussion about font-subsetting can be found in [this GitHub issue](https://github.com/QuestPDF/QuestPDF/issues/31). Moreover, **Pietervdw** (thank you for your help!) created [a detailed instruction](https://github.com/QuestPDF/QuestPDF/issues/31#issuecomment-1018476317) showing how to perform manual font-subsetting.
:::

::: tip
I am planning to implement automated font-subsetting in QuestPDF. Therefore, in the future, this problem will be properly solved. However, taking into account the complexity of this domain, I am planning to focus on other library aspects first. Stay tuned!
:::

## Implementing charts

There are many ways on how to implement charts in the QuestPDF documents. By utilizing the `Canvas` element and SkiaSharp-compatible charting libraries, it is possible to achieve vector charts.

Please analyse this simple example which utilizes the `microcharts` library ([nuget site](https://www.nuget.org/packages/Microcharts/)):

```csharp
// prepare data
var entries = new[]
{
    new ChartEntry(212)
    {
        Label = "UWP",
        ValueLabel = "112",
        Color = SKColor.Parse("#2c3e50")
    },
    new ChartEntry(248)
    {
        Label = "Android",
        ValueLabel = "648",
        Color = SKColor.Parse("#77d065")
    },
    new ChartEntry(128)
    {
        Label = "iOS",
        ValueLabel = "428",
        Color = SKColor.Parse("#b455b6")
    },
    new ChartEntry(514)
    {
        Label = "Forms",
        ValueLabel = "214",
        Color = SKColor.Parse("#3498db")
    }
};

// draw chart using the Canvas element
.Column(column =>
{
    var titleStyle = TextStyle
        .Default
        .Size(20)
        .SemiBold()
        .Color(Colors.Blue.Medium)

    column
        .Item()
        .PaddingBottom(10)
        .Text("Chart example")
        .Style(titleStyle);
    
    column
        .Item()
        .Border(1)
        .ExtendHorizontal()
        .Height(300)
        .Canvas((canvas, size) =>
        {
            var chart = new BarChart
            {
                Entries = entries,

                LabelOrientation = Orientation.Horizontal,
                ValueLabelOrientation = Orientation.Horizontal,
                
                IsAnimated = false,
            };
            
            chart.DrawContent(canvas, (int)size.Width, (int)size.Height);
        });
});
```

This is a result:

![example](./images/patterns-and-practices/chart.png =400x)

## Style

### Color definitions

In QuestPDF, multiple element expect color as part of the configuration. Similarly to HTML, there are a couple of supported formats:
1. Standard format: `RRGGBB` OR `#RRGGBB`
2. Shorthand format: `RGB` or `#RGB`, e.g. `#123` is an equivalent to `#112233`
3. Alpha support: `AARRGGBB` or `#AARRGGBB`
4. Shorthand alpha format: `ARGB` or `#ARGB`

### Material Design colors

You can access any color defined in the Material Design colors set. Please find more details [on the official webpage](https://material.io/design/color/the-color-system.html).

```csharp
// base colors:
Colors.Black
Colors.White
Colors.Transparent

// colors with medium brightness:
Colors.Green.Medium;
Colors.Orange.Medium;
Colors.Blue.Medium;

// darken colors:
Colors.Blue.Darken4
Colors.LightBlue.Darken3
Colors.Indigo.Darken2
Colors.Brown.Darken1

// lighten colors:
Colors.Pink.Lighten1
Colors.Purple.Lighten2
Colors.Teal.Lighten3
Colors.Cyan.Lighten4
Colors.LightGreen.Lighten5

// accent colors:
Colors.Lime.Accent1
Colors.Yellow.Accent2
Colors.Amber.Accent3
Colors.DeepOrange.Accent4
```

Example usage:

```csharp
var colors = new[]
{
    Colors.Green.Darken4,
    Colors.Green.Darken3,
    Colors.Green.Darken2,
    Colors.Green.Darken1,
    
    Colors.Green.Medium,
    
    Colors.Green.Lighten1,
    Colors.Green.Lighten2,
    Colors.Green.Lighten3,
    Colors.Green.Lighten4,
    Colors.Green.Lighten5,
    
    Colors.Green.Accent1,
    Colors.Green.Accent2,
    Colors.Green.Accent3,
    Colors.Green.Accent4,
};

container
    .Padding(25)
    .Height(100)
    .Row(row =>
    {
        foreach (var color in colors)
            row.RelativeItem().Background(color);
    });
```

![example](./images/patterns-and-practices/material-colors.png =450x)

### Basic fonts

The library offers a list of simple and popular fonts.
```csharp
Fonts.Calibri
Fonts.Candara
Fonts.Arial
// and more...
```

Example:

```csharp
var fonts = new[]
{
    Fonts.Calibri,
    Fonts.Candara,
    Fonts.Arial,
    Fonts.TimesNewRoman,
    Fonts.Consolas,
    Fonts.Tahoma,
    Fonts.Impact,
    Fonts.Trebuchet,
    Fonts.ComicSans
};

container.Padding(25).Grid(grid =>
{
    grid.Columns(3);

    foreach (var font in fonts)
    {
        grid.Item()
            .Border(1)
            .BorderColor(Colors.Grey.Medium)
            .Padding(10)
            .Text(font)
            .FontFamily(font).FontSize(16);
    }
});
```

![example](./images/patterns-and-practices/defined-fonts.png =500x)

## Prototyping

### Text

It is a very common scenario when we know how the document layout should look like, however, we do not have appropriate data to fill it. The Quest PDF library provides a set of helpers to generate random text of different kinds:

```csharp
using QuestPDF.Helpers;

Placeholders.LoremIpsum();
Placeholders.Label();
Placeholders.Sentence();
Placeholders.Question();
Placeholders.Paragraph();
Placeholders.Paragraphs();

Placeholders.Email();
Placeholders.Name();
Placeholders.PhoneNumber();

Placeholders.Time();
Placeholders.ShortDate();
Placeholders.LongDate();
Placeholders.DateTime();

Placeholders.Integer();
Placeholders.Decimal();
Placeholders.Percent();
```

### Colors

You can access a random color picked from the Material Design colors set. Colors are returned as text in the HEX format.

```csharp
// bright color, lighten-2
Placeholders.BackgroundColor();

// medium
Placeholders.Color();
```

Example usage to create a colorful matrix:

```csharp
.Padding(25)
.Grid(grid =>
{
    grid.Columns(5);
    
    Enumerable
        .Range(0, 25)
        .Select(x => Placeholders.BackgroundColor())
        .ToList()
        .ForEach(x => grid.Item().Height(50).Background(x));
});
```

![example](./images/patterns-and-practices/random-colors.png =300x)

### Image

Use this simple function to generate random image with required size:

```csharp
// both functions return a byte array containing a JPG file
Placeholders.Image(400, 300);
Placeholders.Image(new Size(400, 300));

// example usage
.Padding(25)
.Width(300)
.AspectRatio(3 / 2f)
.Image(Placeholders.Image);
```

![example](./images/patterns-and-practices/image-placeholder.png =350x)

## Achieving different header/footer on the first page
It is a common requirement to have a special header on the first page on your document. Then all consecutive pages should have a normal header. This requirement can be easily achieved by using the `ShowOnce` and tge `SkipOnce` elements, like so: 

```csharp{8-9}
container.Page(page =>
{
    page.Size(PageSizes.A6);
    page.Margin(30);
    
    page.Header().Column(column =>
    {
        column.Item().ShowOnce().Background(Colors.Blue.Lighten2).Height(60);
        column.Item().SkipOnce().Background(Colors.Green.Lighten2).Height(40);
    });
    
    page.Content().PaddingVertical(10).Column(column =>
    {
        column.Spacing(10);

        foreach (var _ in Enumerable.Range(0, 13))
            column.Item().Background(Colors.Grey.Lighten2).Height(40);
    });
    
    page.Footer().AlignCenter().Text(text =>
    {
        text.DefaultTextStyle(x => x.Size(16));
        
        text.CurrentPageNumber();
        text.Span(" / ");
        text.TotalPages();
    });
});
```

The code above produces the following results:

![example](./images/patterns-and-practices/special-header-on-first-page-1.png =297x)
![example](./images/patterns-and-practices/special-header-on-first-page-2.png =297x)
![example](./images/patterns-and-practices/special-header-on-first-page-3.png =297x)

::: tip
Please notice that you can use the `SkipOnce` and `ShowOnce` elements multiple times to achieve more complex requirements. For example: 
1) `.SkipOnce().ShowOnce()` displays child element only on the second page. 
2) `.SkipOnce().SkipOnce()` displays child element starting at third page.
3) `.ShowOnce().SkipOnce()` displays nothing (invocation order is important!).
:::

## Exceptions

During the development process, you may encounter different issues connected to the PDF document rendering process.
It is important to understand potential sources of such exceptions, their root causes and how to fix them properly.
In the QuestPDF library, all exceptions have been divided into three groups:

###  DocumentComposeException

This exception may occur during the document composition process. When you are using the fluent API to compose various elements together to create the final layout. Taking into account that during such process you are interacting with report data, using conditions, loops and even additional methods, all those operations may be a source of potential exceptions. All of them have been grouped together in this exception type.

### DocumentDrawingException

This exception occurs during the document generation process - when the generation engine translates elements tree into proper drawing commands. If you encounter such an exception, please contact with us - it is most probably a bug that we want to fix for you! Additionally, if you are using custom components, all exceptions thrown there are going to bubble up with this type of exception.

### DocumentLayoutException

This exception may be extremely hard to fix because it happens for valid document trees which enforce constraints that are impossible to meet. For example, when you try to draw a rectangle bigger than available space on the page, the rendering engine is going to wrap the content in a hope that on the next page there would be enough space. Generally, such wrapping behaviour is happening all the time and is working nicely - however, there are cases when it can lead to infinite wrapping. When a certain document length threshold is passed, the algorithm stops the rendering process and throws this exception. In such case, please revise your code and search for indivisible elements requiring too much space.

This exception provides an additional element column. It shows which elements have been rendered when the exception was thrown. Please analyse the example below:

```csharp{2,8}
.Padding(10)
.Width(100)
.Background(Colors.Grey.Lighten3)
.DebugPointer("Example debug pointer")
.Column(x =>
{
    x.Item().Text("Test");
    x.Item().Width(150); // requires 150pt width where only 100pt is available
});
```

Below is the element trace returned by the exception. Nested elements (children) are indented. Sometimes you may find additional, library-specific elements. 

Additional symbols are used to help you find the problem cause:
- 🌟 - represents special elements, e.g. page header/content/footer or the debug pointer element,
- 🔥 - represents an element that causes the layout exception. Follow the symbol deeper to find the root cause.

```
🔥 Page content 🌟
------------------
Available space: (Width: 500, Height: 360)
Requested space: PartialRender (Width: 120,00, Height: 20,00)

    🔥 Background
    -------------
    Available space: (Width: 500, Height: 360)
    Requested space: PartialRender (Width: 120,00, Height: 20,00)
    Color: #ffffff

        🔥 Padding
        ----------
        Available space: (Width: 500, Height: 360)
        Requested space: PartialRender (Width: 120,00, Height: 20,00)
        Top: 10
        Right: 10
        Bottom: 10
        Left: 10

            🔥 Constrained
            --------------
            Available space: (Width: 480, Height: 340)
            Requested space: PartialRender (Width: 100,00, Height: 0,00)
            Min Width: 100
            Max Width: 100
            Min Height: -
            Max Height: -

                🔥 Background
                -------------
                Available space: (Width: 100, Height: 340)
                Requested space: PartialRender (Width: 0,00, Height: 0,00)
                Color: #eeeeee

                    🔥 Example debug pointer 🌟
                    ---------------------------
                    Available space: (Width: 100, Height: 340)
                    Requested space: PartialRender (Width: 0,00, Height: 0,00)

                        🔥 Column
                        --------
                        Available space: (Width: 100, Height: 340)
                        Requested space: PartialRender (Width: 0,00, Height: 0,00)

                            🔥 Padding
                            ----------
                            Available space: (Width: 100, Height: 340)
                            Requested space: PartialRender (Width: 0,00, Height: 0,00)
                            Top: 0
                            Right: 0
                            Bottom: -0
                            Left: 0

                                🔥 BinaryColumn
                                --------------
                                Available space: (Width: 100, Height: 340)
                                Requested space: PartialRender (Width: 0,00, Height: 0,00)

                                    🔥 Constrained
                                    --------------
                                    Available space: (Width: 100, Height: 340)
                                    Requested space: Wrap
                                    Min Width: 150
                                    Max Width: 150
                                    Min Height: -
                                    Max Height: -
```
