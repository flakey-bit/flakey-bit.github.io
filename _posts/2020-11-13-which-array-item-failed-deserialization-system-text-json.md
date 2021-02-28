---
id: 2216
title: 'Which array item failed deserialization? &#8211; System.Text.Json'
date: 2020-11-13T07:27:55+10:00
author: eddiewould
layout: post
guid: https://eddiewould.com/?p=2216
permalink: /2020/11/13/which-array-item-failed-deserialization-system-text-json/
spay_email:
  - ""
categories:
  - Uncategorized
---
A recipe for capturing the JSON of the particular array item that failed deserialization with `System.Text.Json`.

When deserializing a response from an external, third-party API it's fairly common to receive a "wrapper" DTO which consists of an array of "item" DTOs and possibly some additional properties. Depending on the API, the list of items could be large and the object graph of each item could be complex.

Assume you get _well-formed_ JSON back from the API (so no mismatched curlies or anything like that) but deserialization _of one the items in the array_ fails for some reason (perhaps an unsupported enum value or a string where a number should go). By default, you'll still get a `JsonException` back from `System.Text.Json` with something along the lines of

`The JSON value could not be converted to Some.Namespace.Containing.Dtos.EntityStatus. Path: $.status | LineNumber: 8 | BytePositionInLine: 25.`

Not very helpful for production support / debugging. The approach I took to solving this was to introduce a custom `JsonConverter` that captures the raw item JSON and attempts to run it through `JsonSerializer.Deserialize<T>()` . If deserialization fails (i.e. a `JsonException` is thrown) then we throw a custom exception (`JsonItemDeserializationException`) containing the _raw JSON_ that failed to deserialize along with the _underlying_ `JsonException` as the _inner_ exception. Implementation of `CapturingJsonConverter<T>` is given below:


```csharp
public class CapturingJsonConverter<T> : JsonConverter<T>
{
    public override T Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        // Capture the JSON from the reader so that we can include it in the thrown exception if deserialization fails.
        if (!JsonDocument.TryParseValue(ref reader, out var document))
        {
            throw new JsonException();
        }

        var json = document.ToJson();

        // Create a copy of the JsonSerializerOptions except with this converter removed
        // to avoid infinite recursion when we deserialize
        var clonedOptions = options.ShallowClone();
        clonedOptions.Converters.Clear();
        foreach (var converter in options.Converters.Where(c => c.GetType() != GetType()))
        {
            clonedOptions.Converters.Add(converter);
        }

        try
        {
            return JsonSerializer.Deserialize<T>(json, clonedOptions);
        }
        catch (JsonException ex)
        {
            throw new JsonItemDeserializationException(typeof(T), json, ex);
        }
    }

    public override void Write(Utf8JsonWriter writer, T value, JsonSerializerOptions options)
    {
        throw new InvalidOperationException();
    }
}

public class JsonItemDeserializationException : Exception
{
    public JsonItemDeserializationException(Type itemType, string json, JsonException jsonException) : base($"Deserialization of {itemType.Name} JSON item failed", jsonException)
    {
        ItemType = itemType;
        Json = json;
    }

    public Type ItemType { get; }

    public string Json { get; }

    public JsonException JsonException => InnerException as JsonException;
}
```

The code is all pretty self-explanatory - the only _slightly tricky_ bit is we need to create a copy of the `JsonSerializerOptions` with *this converter* removed. If we didn't do that, we'd cause infinite recursion (because the call to `Serializer.Deserialize` would end up using the same converter from the options).

To use the converter, simply add it to the options e.g.

```csharp
private static JsonSerializerOptions BuildJsonSerializerOptions()
{
    return new JsonSerializerOptions
    {
        Converters =
        {
            new JsonStringEnumConverter(),
            new CapturingJsonConverter<MyEntityDto>()
        }
    };
}
```

And at the call site use `JsonSerializer.Deserialize` as usual, but be prepared for *both* `JsonException` and `JsonItemDeserializationException`.

```csharp
try
{
    var wrapper = JsonSerializer.Deserialize<ResponseWrapperDto>(responseJson, serializerOptions);
    return wrapper.Items;
}
catch (JsonItemDeserializationException ex)
{
    Log
        .ForContext("ItemJson", ex.Json)
        .Error(ex, "Failed to deserialize item of type {ItemType}", ex.ItemType);

    throw;
}
catch (JsonException)
{
    // Malformed JSON? In practice would probably omit this catch block
    throw;
}
```

The implementation makes use of a couple of extension methods. Those are given for completeness:

```csharp
public static class JsonDocumentExtensions
{
    public static string ToJson(this JsonDocument document)
    {
        using var stream = new MemoryStream();
        using Utf8JsonWriter writer = new Utf8JsonWriter(stream, new JsonWriterOptions {Indented = true});
        document.WriteTo(writer);
        writer.Flush();

        return Encoding.UTF8.GetString(stream.GetBuffer());
    }
}

public static class JsonSerializerOptionsExtensions
{
    public static JsonSerializerOptions ShallowClone(this JsonSerializerOptions options)
    {
        var cloned = new JsonSerializerOptions
        {
            AllowTrailingCommas = options.AllowTrailingCommas,
            DefaultBufferSize = options.DefaultBufferSize,
            Encoder = options.Encoder,
            DictionaryKeyPolicy = options.DictionaryKeyPolicy,
            IgnoreNullValues = options.IgnoreNullValues,
            IgnoreReadOnlyProperties = options.IgnoreReadOnlyProperties,
            MaxDepth = options.MaxDepth,
            PropertyNamingPolicy = options.PropertyNamingPolicy,
            PropertyNameCaseInsensitive = options.PropertyNameCaseInsensitive,
            ReadCommentHandling = options.ReadCommentHandling,
            WriteIndented = options.WriteIndented
        };

        foreach (var converter in options.Converters)
        {
            cloned.Converters.Add(converter);
        }

        return cloned;
    }
}
```

It's probably worth calling out *not* to use this in performance-critical code (by reading the `Utf8JsonReader` into a `JsonDocument` we're no-doubt removing a bunch of the low-level-string handling memory optimizations).
