# The .NET Show Episode 2 

###### Building a Mobile Podcast App Part 1

## Overview

In the next few episodes of The .NET Show, I will build a mobile podcast app for my podcast, .NET Rocks! using Xamarin Forms. We had hired an outside company to build our podcast app years ago, but they are no longer available. 

Since this is the first real Xamarin Forms project I have featured on The .NET Show, I would be remiss if I didn't talk about how to set up your Xamarin Forms development environment.

## Getting Started with Xamarin Forms

Expect to spend a few hours tracking down issues. If everything works out of the box, great! But, there are some classic issues people run into with Hyper-V, phone drivers, and even bad USB cables. 

This document is where you should start when attempting Xamarin Forms for the first time. 

https://docs.microsoft.com/en-us/xamarin/get-started/installation/?pivots=windows

### Debugging with Android Emulator

The easiest way to debug and test is using an Android Emulator. If you want to use the Android emulator, you really should enable Hyper-V support in Windows, otherwise debugging will take FOREVER. I went down the rabbit hole of figuring out if I could debug in my standard Azure VM, which is Windows 10 Professional. The answer, sadly, is no. If you want to use the emulator in a VM, you have to have an OS that supports Second Level Address Translation (SLAT) - and that's a Server SKU. You can read the latest about this issue at https://docs.microsoft.com/en-us/xamarin/android/get-started/installation/android-emulator/hardware-acceleration?pivots=windows

Here's a good resource that shows you how to enable Hyper-V on your local Windows 10 machine:

https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v

### Debugging with an Android Phone

This document goes through how to configure your android phone to work as a debugging device:

https://docs.microsoft.com/en-us/xamarin/android/get-started/installation/set-up-device-for-development

TLDR: You need to enable your phone for debugging, and then install a USB driver in Windows. 

#### Android Screen Mirroring

If you're doing a demo and want to show the Android phone screen on Windows, try the tool/app combination MobiZen (https://www.mobizen.com/). It's a deprecated product, but it still works. It's free, and doesn't interfere with your phone drivers.

### Debugging with an iPhone

#### Hot Restart

The easiest way to use an iPhone is using Xamarin Hot Restart.

https://docs.microsoft.com/en-us/xamarin/xamarin-forms/deploy-test/hot-restart

Make sure you have the latest version of Visual Studio 2019. Xamarin Hot Restart is enabled by default.

You also need to install iTunes, and make sure your PC recognizes your iPhone when you plug it in. Lots of people dismiss the possibility of using a bad cable. Here's a great resource to help you figure out why your iPhone isn't being recognized: https://support.apple.com/en-us/HT204095

I had an issue after installing an application that showed my iPhone screen on my computer. I found that I didn't have the latest drive for the iPhone. I went into Control Panel Device Manager, and saw a generic USB win32 item when I plugged in the iPhone. I right-clicked and searched for a new driver. Windows found the correct Apple driver, and everything worked after that.

You also must create an application in the iOS Developer center. https://developer.apple.com/programs You have to pay $100/year to enroll in the Apple program. 

#### iPhone Screen Mirroring

I use a free tool/app called LetsView (https://letsview.com/windows) to show my iPhone screen. The only drawback is that the image doesn't scale on the screen, so you get what you get, but it works and (again) it doesn't interfere with your existing drivers.

### Using an iPhone Simulator

If you want to use an iPhone or iPad simulator, you'll need to have a mac connected to your network that has Xcode installed on it. Here's a how-to:

https://docs.microsoft.com/en-us/xamarin/ios/get-started/installation/windows/connecting-to-mac/

## Create the MobileDnr App

MobilDnr is an app for Android and iOS that downloads, caches, and plays .NET Rocks! episodes. We're going to build it together piece by piece, taking it all the way to deployment in the app stores.

### Step 1: Create the app, play audio

The first step is to create the application, make sure we can test both the Android and the iOS versions, and implement code to download and play an mp3 file.

Create a blank Xamarin Forms mobile application called `MobileDnr`. Use the *Blank* template, not the *Flyout* or *Tabbed* template.

Next, add these packages to the solution, in all three projects.

```c#
Refractored.MvvmHelpers
Plugin.MediaManager.Forms
```
Add to *MainActivity.cs* in `OnCreate()`

```c#
CrossMediaManager.Current.Init(this);
```

Add to *AppDelegate.cs* in `FinishedLaunching()`

```c#
CrossMediaManager.Current.Init();
```

Add the following class to UI project:

*MainViewModel.cs*

```c#
using System;
using System.Collections.Generic;
using System.Text;
using System.Runtime.CompilerServices;
using System.ComponentModel;
using MvvmHelpers;

namespace MobileDnr
{
    public class MainViewModel : BaseViewModel
    {
        string url = "https://pwop6300.blob.core.windows.net/mtfb/01-Garde1.mp3";
    }
}
```

Replace `MainPage.xaml` with this

```xaml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml" 
             xmlns:viewmodel="clr-namespace:MobileDnr"
             x:DataType="viewmodel:MainViewModel"
             x:Class="MobileDnr.MainPage">
    <ContentPage.BindingContext>
        <viewmodel:MainViewModel />
    </ContentPage.BindingContext>
    <StackLayout>
        <StackLayout HorizontalOptions="Center" Margin="0,200,0,0">
            <Label Text="Let's try to play some Audio" />
            <Button Text="Play" Command="{Binding Play}" />
        </StackLayout>
    </StackLayout>

</ContentPage>
```

Just as James Montemagno showed us in [Episode 1 of The .NET Show](https://www.youtube.com/watch?v=EGR-7ft34sM), it's easy to use a ViewModel in Xamarin Forms. I've created a `viewmodel` XAML namespace pointing to our `MobileDnr` code namespace, and I've also added a `<ContentPage.BindingContext/>` section where we bind to the `MainViewModel`.

Generate the command by hovering over the words `{Binding Play}` in the button definition.

In *MainViewModel.cs* change `PerformPlay()` to `async` and add code to play the url:

```c#
private async Task PerformPlay()
{
    await CrossMediaManager.Current.Play(url);
}
```

Notice that there's an issue on line 26, because Commands can not be async.

However, the `MvvmHelpers` library adds an `AsyncCommand`. Change the `Command` to an `AsyncCommand` like so:

```c#
play = new AsyncCommand(PerformPlay);
```

You'll need this:

```
using MvvmHelpers.Commands;
```

Select a platform to run by making it the startup project, and run.

### Step 2: Add a Stop button

Before we add a stop button, we need to know whether the app is playing or not. That needs to be expressed as a boolean property in the ViewModel so we can bind to it. 

We also need to bind to the *inverse* of this boolean property. For that, we need a value converter.

Add the following class to the `MobileDnr` project:

*InverseBoolConverter.cs*

```c#
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Text;
using Xamarin.Forms;

namespace MobileDnr
{
    public class InverseBoolConverter : IValueConverter
    {
        public object Convert(object value, Type targetType, 
            object parameter, CultureInfo culture)
        {
            return !((bool)value);
        }

        public object ConvertBack(object value, Type targetType,
            object parameter, CultureInfo culture)
        {
            return value;
        }
    }
}
```

Let's add an `IsPlaying` property to *MainViewModel.cs*:

```c#
private bool isPlaying;
public bool IsPlaying
{
    get
    {
        return isPlaying;
    }
    set
    {
        SetProperty(ref isPlaying, value);
    }
}
```

Next, let's add a `Stop` command to the *MainViewModel.cs*:

```c#
private ICommand stop;

public ICommand Stop
{
    get
    {
        if (stop == null)
        {
            stop = new AsyncCommand(PerformStop);
        }
        return stop;
    }
}

protected async Task PerformStop()
{
    IsPlaying = false;
    await CrossMediaManager.Current.Stop();
}
```

Now we need to modify our `PerformPlay()` method to set the `IsPlaying` property to true;

```c#
private async Task PerformPlay()
{
    IsPlaying = true;
    await CrossMediaManager.Current.Play(url);
}
```

With that plumbing in place, let's modify *MainPage.xaml* to create a `Stop` button using the `IsPlaying` property to determine visibility.

```xaml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml" 
             xmlns:local="clr-namespace:MobileDnr"
             x:DataType="local:MainViewModel"
             x:Class="MobileDnr.MainPage">
    <ContentPage.BindingContext>
        <local:MainViewModel />
    </ContentPage.BindingContext>
    <ContentPage.Resources>
        <local:InverseBoolConverter x:Key="InverseBoolConverter"/>
    </ContentPage.Resources>
    <StackLayout>
        <StackLayout HorizontalOptions="Center" Margin="0,200,0,0">
            <Label Text="Let's try to play some Audio" />
            <Button Text="Play" 
                    IsVisible="{Binding IsPlaying, 
                        Converter={StaticResource InverseBoolConverter}}"
                    Command="{Binding Play}" />
            <Button Text="Stop" 
                    IsVisible="{Binding IsPlaying}" 
                    Command="{Binding Stop}" />
        </StackLayout>
    </StackLayout>
</ContentPage>
```

I've made a few changes here. I've changed the `viewmodel` namespace to `local`, I've added a `<ContentPage.Resources>` section where I've defined the `InverseBoolConverter` resource to point to our *InverseBoolConverter.cs* class, I've added a `Stop` button, and I've bound both the buttons' `IsVisible` properties to the `IsPlaying` property of the ViewModel. I've also used the `InverseBoolConverter` in the binding for the `Play` button. 

The result is that when `IsPlaying` is false, the `Play` button will be visible and the `Stop` button will not be visible. When `IsPlaying` is true (after you press the `Play` button), the `Play` button will NOT be visible, and the `Stop` button will.

The UI isn't pretty, but this is basic XAML stuff here. I have no doubt that even with MAUI, you'll still need to make Value Converters for stuff like this.

When I run the app on my Android phone, it looks like this:

![image-20210428130407839](https://pwop6300.blob.core.windows.net/thedotnetshow/Shot1.png)

When I press the `Play` button, after a pause for downloading, I see this:

![image-20210428130456093](https://pwop6300.blob.core.windows.net/thedotnetshow/Shot2.png)

Pressing the `Stop` button stops the audio and reverts the UI back to it's original state.

### Step 3: Add Remaining Time Display

This feature will show us how many minutes and seconds are left to play, updating every second.

First, let's add a `CurrentStatus` string property to *MainViewModel.cs*

```c#
private string currentStatus;
public string CurrentStatus { get => currentStatus; set => SetProperty(ref currentStatus, value); }
```

Next, let's handle the `CrossMediaManager`'s  `PositionChanged` event to so we can update the status as the audio is playing:

```c#
private void Current_PositionChanged(object sender, MediaManager.Playback.PositionChangedEventArgs e)
{
    TimeSpan currentMediaPosition = CrossMediaManager.Current.Position;
    TimeSpan currentMediaDuration = CrossMediaManager.Current.Duration;
    TimeSpan TimeRemaining = currentMediaDuration.Subtract(currentMediaPosition);
    if (IsPlaying)
        CurrentStatus = $"Time Remaining: {TimeRemaining.Minutes:D2}:{TimeRemaining.Seconds:D2}";
}
```

Let's also handle the `CrossMediaManager`'s  `MediaItemFinished` event to so we can update things when the audio is done playing:

```c#
private void Current_MediaItemFinished(object sender, MediaManager.Media.MediaItemEventArgs e)
{
    CurrentStatus = "";
    IsPlaying = false;
}
```

We now need to wire up these event handlers in a constructor:

```c#
public MainViewModel()
{
    CrossMediaManager.Current.PositionChanged += Current_PositionChanged;
    CrossMediaManager.Current.MediaItemFinished += Current_MediaItemFinished;
}
```

One more change in the ViewModel. When `Play` is pressed, while the mp3 file is downloading, let's change the `CurrentStatus` to "Downloading...":

```c#
private async Task PerformPlay()
{
    IsPlaying = true;
    CurrentStatus = "Downloading...";
    await CrossMediaManager.Current.Play(url);
}
```

We'll also make sure to clear `CurrentStatus` after hitting the `Stop` button:

```c#
protected async Task PerformStop()
{
    IsPlaying = false;
    CurrentStatus = "";
    await CrossMediaManager.Current.Stop();
}
```

Now, in *MainPage.xaml* let's add a label underneath the buttons:

```xaml
<Label IsVisible="{Binding IsPlaying}" Text="{Binding CurrentStatus}" />
```

That should do it! See how easy it is to add features once you are set up and running!

Here's a shot of my Android phone after clicking `Play` but before playing starts:

![image-20210428133500172](https://pwop6300.blob.core.windows.net/thedotnetshow/Shot3.png)

And then after it starts playing:

![image-20210428133513264](https://pwop6300.blob.core.windows.net/thedotnetshow/Shot4.png)

There's obviously a lot more to this app than playing an mp3 file. In the next section we will cache the mp3 file so it will play even if offline, and then we'll get a list of mp3 files (with their metadata) from an RSS feed.

