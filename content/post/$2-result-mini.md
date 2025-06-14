+++
author = "katlego modupi"
title = "result.mini"
date = "2025-05-25"
description = "yet another result type for c#"
tags = [
    "dotnet", "c#", "exceptions", "error handling"
]
+++

Writing yet another result type for c#.

[![github repo](https://img.shields.io/badge/result.mini-gray?logo=github)](https://github.com/kat-lego/result-mini)
[![nuget](https://img.shields.io/badge/result.mini-gray?logo=nuget)](https://www.nuget.org/packages/Result.Mini)
<!--more-->

I decided to write yet another result type for c#. A generic type used to encapsulate the result of a
function call as either a success or failure.

## why?
This is simply a ploy to talk about one of the many topics I find interesting (error handling), as well as showcasing some cool features of c#.

## requirements
Imagine we have a function that makes a call to a weather api, which uses an HTTP client. The HTTP 
client may throw any one of several exceptions. The GetWeather function may want to catch these exceptions and re-package them in a way that makes sense in the application's context. For example, 
it might throw a RetryableErrorException or FatalErrorException if that's all that matters to the 
function's consumers.

```cs
public async Task<WeatherData> GetWeatherAsync()
{
    try
    {
        var response = await client.GetAsync("https://fakeweather.com/forecast");

        var content = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<WeatherData>(content) 
            ?? throw new FatalErrorException("Failed to parse weather data.", null);
    }
    catch (HttpRequestException ex)
    {
        throw new RetryableErrorException(
            "Network error occurred while calling weather API.", ex);
    }
    catch (JsonException ex)
    {
        throw new FatalErrorException(
            "Weather data response was malformed.", ex);
    }
}
```

This abstracts away the internals of the GetWeather function, the GetWeather function may as well
be getting its information from a file that has to be updated by the intern with his own sets of
problems to deal with. I like this.

Suppose we have a Background service called SkyDivingPricingRevisionWorker which is supposed to
call GetWeather and CalculatePrice with the output from the weather. If there is an error that
is not retryable the service should call a SendPagerAlert function.

```cs
public class SkyDivingPricingRevisionWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var weather = await weatherService.GetWeatherAsync();
                var price = CalculatePrice(weather);
                logger.LogInformation("Calculated new price: {Price}", price);
            }
            catch (RetryableErrorException ex)
            {
                logger.LogWarning(ex, 
                    "Retryable error while retrieving weather data. Will try again later.");
            }
            catch (FatalErrorException ex)
            {
                logger.LogError(ex, 
                    "Fatal error while retrieving weather data.");
                SendPagerAlert(
                    "Skydiving pricing failed due to weather service error.");
            }

            await Task.Delay(TimeSpan.FromMinutes(10), stoppingToken);
        }
    }
}
```

In this example, we are now using exceptions to control logic flow. It is sufficient now, however,
try/catch blocks aren't as versatile as if statements to control flow. For example:

- nested try-catch blocks are more cursed than nested if statements.
- can't do an inverted if/else to exit early

In this stack, we will swap this out for the result pattern to look like this.

`GetWeather`
```cs

public async Task<Result<WeatherData>> GetWeatherAsync()
{
    try
    {
        var response = await client.GetAsync("https://fakeweather.com/forecast");

        var content = await response.Content.ReadAsStringAsync();
        var data = JsonSerializer.Deserialize<WeatherData>(content);
        if(data != null)
        {
            return data
        }
        return new Error(Error.RetryableError, "Failed to parse weather data.");
    }
    catch (HttpRequestException ex)
    {
        return new Error(Error.RetryableError,
            "Network error occurred while calling weather API.");
    }
    catch (JsonException ex)
    {
        return new Error(Error.FatalError, "Weather data response was malformed.");
    }
}
```

`SkyDivingPricingRevisionWorker`
```cs
public class SkyDivingPricingRevisionWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var (weather, error) = await weatherService.GetWeatherAsync();

            if(error != null and error.Code == Error.RetryableException)
            {
                logger.LogWarning(ex,
                    "Retryable error while retrieving weather data. Will try again later.");
                continue;
            }

            if(error != null and error.Code == Error.RetryableException)
            {
                logger.LogError(ex,
                    "Fatal error while retrieving weather data.");
                SendPagerAlert(
                    "Skydiving pricing failed due to weather service error.");
                continue;
            }

            var price = CalculatePrice(weather);

            logger.LogInformation("Calculated new price: {Price}", price);

            await Task.Delay(TimeSpan.FromMinutes(10), stoppingToken);
        }
    }
}
```

This approach makes the control flow clearer and more predictable. Instead of catching exceptions,
we handle results directly—making the code easier to follow, test, and extend.

## the stack

Here is the Result type:

```cs
public struct Result<T>
{
    private readonly T? data;
    private readonly IEnumerable<Error> errors = [];

    private Result(T data)
    {
        this.data = data;
    }

    private Result(IEnumerable<Error> errors)
    {
        errors = errors;
        data = default;
    }

    private Result(Error error)
    {
        errors = [error];
        data = default;
    }

    public static implicit operator Result<T>(T data)
    {
        return new Result<T>(data);
    }

    public static implicit operator Result<T>(Error error)
    {
        return new Result<T>(error);
    }

    public static implicit operator Result<T>(Error[] errors)
    {
        return new Result<T>(errors);
    }

    public void Deconstruct(out T? data, out IEnumerable<Error> errors)
    {
        data = this.data;
        errors = this.errors;
    }

}
```

We will go through some key features to achieve what we saw in the example reference.

### implicit operators

```cs
public static implicit operator Result<T>(T data)
{
    return new Result<T>(data);
}

public static implicit operator Result<T>(Error error)
{
    return new Result<T>(error);
}

public static implicit operator Result<T>(Error[] errors)
{
    return new Result<T>(errors);
}
```

Implicit operators allow us to automatically convert an instance of T or Error when you assign or return to a type Result<T>

### Deconstructor

```cs
public void Deconstruct(out T? data, out IEnumerable<Error> errors)
{
    data = this.data;
    errors = this.errors;
}
```

This allows us to unpack a result object into a tuple (data, errors), which in turn allows us to have
a go-style way of error handling.

And that completes the stack. Ironically, you can just have your functions return the tuple (T,
errors) to begin with.

