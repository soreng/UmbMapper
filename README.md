# <img src="https://raw.githubusercontent.com/JimBobSquarePants/UmbMapper/develop/build/assets/logo/umbmapper-64.png" width="52" height="52" alt="UmbMapper Logo"/> UmbMapper

This repository contains a fast, simple to use, convention based published content mapper for Umbraco.

## What does it do?

UmbMapper maps `IPublishedContent` instances from the Umbraco Published Content Cache to strongly typed classes. It does so in an efficient manner with very little overhead.

So far it's made up of the following libraries

- [**UmbMapper**](https://www.nuget.org/packages/UmbMapper) - The main mapping library, Maps all default Umbraco dataTypes and almost anything that uses a `PropertyValueConverter` to POCO equivalents.
- [**UmbMapper.ArcheType**](https://www.nuget.org/packages/UmbMapper.ArcheType) - Allows the mapping of ArcheType models to POCO equivalents.
- [**UmbMapper.NuPickers**](https://www.nuget.org/packages/UmbMapper.NuPickers) - Allows the mapping of NuPicker models to POCO equivalents.
- [**UmbMapper.PublishedContentModelFactory**](https://www.nuget.org/packages/UmbMapper.PublishedContentModelFactory) - Allows the mapping of models using the Umbraco PublishedContentFactory.

## Consuming The Libraries

Nightlies are available on [Myget](https://www.myget.org/gallery/umbmapper) with battle tested releases available on [Nuget](https://www.nuget.org/packages/UmbMapper).

Samples project login

- Username : admin
- Password : umbmapper!

## The API

> *A POCO should be exactly that.* - Rudyard Kipling

Models are clean without any knowledge of the Umbraco backoffice. They require no additional attribution to determine mapping logic.

Here's an example class, nothing fancy!

``` csharp
public class LazyPublishedItem
{
    public virtual int Id { get; set; }

    public virtual string Name { get; set; }

    public virtual string Slug { get; set; }

    public virtual DateTime CreateDate { get; set; }

    public virtual DateTime UpdateDate { get; set; }

    public virtual PlaceOrder PlaceOrder { get; set; }

    public virtual IPublishedContent PublishedInterfaceContent { get; set; }

    public virtual MockPublishedContent PublishedContent { get; set; }

    public virtual ImageCropDataSet Image { get; set; }

    public virtual PublishedItem Child { get; set; }
}
```

You'll have noticed the `virtual` keyword there. It's only required when we want to lazy map a property so it's not essential. (more on that later)

The mapping behaviour is controlled by creating a mapping class that inherits from `MapperConfig<T>` where `T` is the class you want to map to.

``` csharp
public class LazyPublishedItemMap : MapperConfig<LazyPublishedItem>
{
    public LazyPublishedItemMap()
    {
        this.AddMap(p => p.Id).AsLazy();
        this.AddMap(p => p.Name).AsLazy();
        this.AddMap(p => p.Slug).MapFromInstance((instance, content) => instance.Name.ToLowerInvariant());
        this.AddMap(p => p.CreateDate).AsLazy();
        this.AddMap(p => p.UpdateDate).SetAlias(p => p.UpdateDate, p => p.CreateDate).AsLazy();
        this.AddMap(p => p.PlaceOrder).SetMapper<EnumPropertyMapper>().AsLazy();
        this.AddMap(p => p.PublishedInterfaceContent).SetMapper<UmbracoPickerPropertyMapper>().AsLazy();
        this.AddMap(p => p.PublishedContent).SetMapper<UmbracoPickerPropertyMapper>().AsLazy();
        this.AddMap(p => p.Image).AsLazy();
        this.AddMap(p => p.Child).SetMapper<UmbracoPickerPropertyMapper>().AsLazy();
    }
}
```

For simpler classes there are more terse mapping methods available

``` csharp
public class LazyPublishedItemMap : UmbMapperConfig<LazyPublishedItem>
{
    public LazyPublishedItemMap()
    {
        this.AddMappings(
            x => x.Id,
            x => x.Name,
            x => x.DocumentTypeAlias,
            x => x.Level).ForEachIndexed((x, i) => x.AsLazy());

        this.AddMappings(
            x => x.SortOrder,
            x => x.CreateDate,
            x => x.UpdateDate).ForEach(x => x.AsLazy());
    }
}
```

Or

``` csharp
public class LazyPublishedItemMap : UmbMapperConfig<LazyPublishedItem>
{
    public LazyPublishedItemMap()
    {
        this.MapAll().ForEachIndexed((x, i) => x.AsLazy());

        this.MapAll().ForEach(x => x.AsLazy());
    }
}
```

**For many classes where no additional mapping logic is required you don't even need an UmbMapperConfig. See *Registering a mapper* below.**

**This is the recommended way to map classes**

What's going on here? 

### Configuration 

The `AddMap()` method and subsequent methods called in the mapper constructor each return a `PropertyMap<T>` where `T` is the class you want to map to. This allows us to use a simple fluent API to configure each property map.

The various mapping configuration options are as follows:

- `AddMap()` Instructs the mapper to map the property.
- `AddMappings()` Instructs the mapper to map the collection of properties.
- `MapAll()` Instructs the mapper to map all the the properties in the class.
- `MapFromInstance()` Instructs the mapper to map from the given `Func<T, IPublishedContent, object>` where `T` is the current object instance.
- `Ignore()` Removes a property from the collection of property maps.
- `SetAlias()` Instructs the mapper what aliases to look for in the document type. The order given is the checking order. Case-insensitive.
- `SetMapper()` Instructs the mapper what specific `IPropertyMapper` implementation to use for mapping the property. All properties are initially automatically mapped using the `UmbracoPickerPropertyMapper`.
- `SetCulture()` Instructs the mapper what culture to use when mapping values. Defaults to the current culture contained withing the `UmbracoContext`.
- `AsRecursive()` Instructs the mapper to recursively traverse up the document tree looking for a value to map.
- `AsLazy()` Instructs the mapper to map the property lazily using dynamic proxy generation.

Available `IPropertyMapper`implementations all inherit from the `PropertyMapperBase` class and are as follows:

- `UmbracoPickerPropertyMapper` The default mapper, maps directly from Umbraco's Published Content Cache via `GetPropertyValue`. Runs automatically.
- `EnumPropertyMapper` Maps to enum values. Can handle both integer and string values.
- `UmbracoPickerPropertyMapper` Maps from all the Umbraco built-in legacy pickers. Not required for any of the pickers from v7.6+
- `DocTypeFactoryPropertyMapper` Allows mapping from mixed `IPublishedContent` sources sharing common properties. Inherits `FactoryPropertyMapperBase`.

These mappers handle most use cases since they rely initially on Umbraco's `PropertyValueConverter` API. Additional mappers can be easily created though. Check the source for examples.

### Calling a Mapper

There are four extension methods that have been added to the `IPublishedContent` interface providing compile-time and run-time mapping variants. Their signatures are as follows:

``` csharp
// Map a collection of a compile-time known type instances.
public static IEnumerable<T> MapTo<T>(this IEnumerable<IPublishedContent> content)
    where T : class

// Map a collection of a run-time known type instances.
public static IEnumerable<object> MapTo(this IEnumerable<IPublishedContent> content, Type type)

// Map a single compile-time known type instance.
public static T MapTo<T>(this IPublishedContent content)
    where T : class

// Map a single run-time known type instance.
public static object MapTo(this IPublishedContent content, Type type)
```

Registering a mapper is as easy as follows


``` csharp
// Add the mapper we created to the registry
UmbMapperRegistry.AddMapper(new LazyPublishedItemMap());

// For simple classes you don't need to create a mapper. 
// The registry will automatically create one based on the default conventional logic.
// Mappers added this way automatically use lazy mapping for `virtual` properties.
UmbMapperRegistry.AddMapperFor<SimpleItem>;
```

## IPublishedContentModelFactory

As of v7.1.4, Umbraco ships with using a default model factory for `IPublishedContent`.
For more information about the [IPublishedContentModelFactory](https://github.com/zpqrtbnk/Zbu.ModelsBuilder/wiki/IPublishedContentModelFactory) please the "Zbu.ModelsBuilder" wiki:

UmbMapper comes with an implementation that can be configured as following.

``` csharp
using UmbMapper.PublishedContentModelFactory;

public class ConfigurePublishedContentModelFactory : ApplicationEventHandler
{
	protected override void ApplicationStarting(UmbracoApplicationBase umbracoApplication, ApplicationContext applicationContext)
	{
		// Important! This has to occur after mapper registration
		var factory = new UmbMapperPublishedContentModelFactory();
		PublishedContentModelFactoryResolver.Current.SetFactory(factory);
	}
}
```

> Note: It's recommended that you use lazy property mapping when using the `UmbMapperPublishedContentModelFactory` as it ensures that any `PropertyValueConverter`implementations 
that have `UmbracoContext` based requirements may fail otherwise.

## Performance

**UmbMapper is blazingly quick**. I've taken all the super fast bits I'd written in the Ditto library and was able to turbo charge them a little more. The underpinning logic is simple also and requires very little run-time work as the rules are already determined at compile-time.

Additional performance boosting can be delivered using lazy mapping.

## Dynamic Proxies and Lazy Mapping

> *Any sufficiently advanced technology is indistinguishable from magic.*  - Gandalf the Grey

If a mapper is configured using the `AsLazy()` instruction and the class contains properties using the `virtual` keyword, the mapper will generate a dynamic proxy class at run-time to represent the type we are mapping to. This class actually inherits our target class and you'll be able to see it by attaching a debugger.

Any properties configured with lazy mapping are not actually mapped until you specifically call the getter on the property. (Via means of `MethodInfo` interception using terribly complicated `Reflection.Emit`) this means that we can map large collections with very limited overheads.