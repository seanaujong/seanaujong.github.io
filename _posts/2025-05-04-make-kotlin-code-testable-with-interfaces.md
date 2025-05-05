---
layout: post
title: Make Kotlin code testable with interfaces
date: 2025-05-04 22:23 -0500
categories: [Intro]
tags: [swe, testing, kotlin]
---

# Before you begin...

Please pull up an IDE with:
- one file for production code and
- one file for JUnit tests

# 1. Testing the time

Let's say we have a class that fetches the current time and does something with it:

```kotlin
class Greeter {
    fun greet(): String {
        val hour = Calendar.getInstance().get(Calendar.HOUR_OF_DAY) // hardcoded
        return if (hour < 12) "Good morning" else "Good evening"
    }
}
```

We want to test the case where `greet` returns "Good morning", AND the case where `greet` returns "Good evening". The problem is that `Calendar.getInstance()` fetches the real system time.

Consider the below example; this unit test will awkwardly fail for half the day!

```kotlin
@Test
fun testMorningGreeting() {
    val greeter = Greeter()
    assertEquals("Good morning", greeter.greet())
}
```

Here's a thought - `greet` doesn't actually care where you get the time from. It just wants **something** that can get the time.

When you get that thought, that's when you need an interface!

```kotlin
interface Clock {
    fun currentHour(): Int
}
```

The `interface Clock` says, "a Clock is anything that gives you the `currentHour`". As long as it's a `Clock`, it should be usable by `Greeter`:

```kotlin
class Greeter(private val clock: Clock) { // just give me a Clock, any Clock!
    fun greet(): String {
        val hour = clock.currentHour()
        return if (hour < 12) "Good morning" else "Good evening"
    }
}
```

You want **something**, **anything** that implements `Clock`. Here's two examples:

```kotlin
class SystemClock : Clock {
    override fun currentHour(): Int {
        return Calendar.getInstance().get(Calendar.HOUR_OF_DAY)
    }
}

class HardcodedClock(private val fixedHour: Int) : Clock {
    override fun currentHour(): Int = fixedHour
}
```

The `SystemClock` is taken from the first code snippet; you would use this for your actual code. On the other hand, the `HardcodedClock` is really useful for unit tests:

```kotlin
@Test
fun testMorningGreeting() {
    val greeter = Greeter(HardcodedClock(9))
    assertEquals("Good morning", greeter.greet())
}

@Test
fun testEveningGreeting() {
    val greeter = Greeter(HardcodedClock(18))
    assertEquals("Good evening", greeter.greet())
}
```

By swapping out the `Clock` implementation, we're able to write consistent, useful unit tests for all scenarios!

# 2. Testing a network call

Network calls can either succeed or fail. You probably have code paths for both cases. Just like how you shouldn't use a real system clock during unit tests, you shouldn't make real network calls either.

Say you have a `WeatherForecaster` that tells you if it is hot today:

```kotlin
class WeatherForecaster() {
    suspend fun isHotToday(): Boolean {
        return try {
            // CnnWeatherForecast is a class that makes a network request to cnn.com
            val weatherApi = CnnWeatherForecast()
            val weather = weatherApi.getCurrentWeather()
            weather.temperatureCelsius > 30.0
        } catch (e: Exception) {
            false
        }
    }
}
```

Like the last example, we can't control if `cnn.com` will give us a temperature above or below 30C. We also can't control if `cnn.com` will give us a network error or not.

Once again, we just want **something** that can get us the weather. We'll take anything that satisfies the `interface WeatherApi`:

```kotlin
data class WeatherResponse(val temperatureCelsius: Double)

interface WeatherApi {
    suspend fun getCurrentWeather(): WeatherResponse
}

// we'll take any WeatherApi implementation, NOT just cnn.com!
class WeatherForecaster(private val weatherApi: WeatherApi) {
    suspend fun isHotToday(): Boolean {
        val weather = weatherApi.getCurrentWeather()
        return weather.temperatureCelsius > 30.0
    }
}
```

In the last example, we literally typed up a `HardcodedClock : Clock` that we could manipulate for our desired output. But what if I told you there were libraries out there that do this for you?

I'll use the `mockk` library this time - `mockk<WeatherApi>()` automagically creates an object that implements `WeatherApi`. On top of that, we'll get helper test functions that **verify we called the weatherApi**:

```kotlin
@ExperimentalCoroutinesApi
class WeatherForecasterTest {

    private val weatherApi = mockk<WeatherApi>()
    private lateinit var forecaster: WeatherForecaster

    @Before
    fun setup() {
        forecaster = WeatherForecaster(weatherApi)
    }

    @Test
    fun `isHotToday returns true when temperature is above 30`() = runTest {
        val hotWeather = WeatherResponse(temperatureCelsius = 35.0)
        coEvery { weatherApi.getCurrentWeather() } returns hotWeather

        val result = forecaster.isHotToday()

        assertTrue(result)
        coVerify(exactly = 1) { weatherApi.getCurrentWeather() }
    }

    @Test
    fun `isHotToday returns false when temperature is below or equal to 30`() = runTest {
        val coolWeather = WeatherResponse(temperatureCelsius = 25.0)
        coEvery { weatherApi.getCurrentWeather() } returns coolWeather

        val result = forecaster.isHotToday()

        assertFalse(result)
        coVerify(exactly = 1) { weatherApi.getCurrentWeather() }
    }

    @Test
    fun `isHotToday returns false when API throws exception`() = runTest {
        coEvery { weatherApi.getCurrentWeather() } throws IOException("No network")

        val result = forecaster.isHotToday()

        assertFalse(result)
        coVerify(exactly = 1) { weatherApi.getCurrentWeather() }
    }
}
```

# Ok, but what if my code uses Spring, or X/Y/Z framework

That's totally fine! From the last two examples, you've learned that your functions/classes should accept **interfaces**. If your code feels untestable, there's probably a part in there that you could separate with an interface. It should be pretty obvious - usually you're interfacing out a clock, a network request, or a database call.

# By the way, you just learned dependency injection

Dependency injection is literally just when you go from this:

```kotlin
class Greeter {
    fun greet(): String {
        val hour = Calendar.getInstance().get(Calendar.HOUR_OF_DAY) // hardcoded
        return if (hour < 12) "Good morning" else "Good evening"
    }
}
```

to this:

```kotlin
class Greeter(private val clock: Clock) { // just give me a Clock, any Clock!
    fun greet(): String {
        val hour = clock.currentHour()
        return if (hour < 12) "Good morning" else "Good evening"
    }
}

interface Clock {
    fun currentHour(): Int
}
```

# Author's note

I struggled with understanding testable code for the longest time. It feels like the popular resources today, although well-intentioned, just don't explain this topic intuitively. As readers, it's too easy to nod our heads to flowery buzzwords. I hope after reading this article, you understand that testable code isn't just some moral highground - it's also just easier to work with.
