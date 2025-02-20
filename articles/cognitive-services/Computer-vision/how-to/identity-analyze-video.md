---
title: "Example: Real-time video analysis - Face"
titleSuffix: Azure Cognitive Services
description: Use the Face service to perform near-real-time analysis on frames taken from a live video stream.
services: cognitive-services
author: SteveMSFT
manager: nitinme

ms.service: cognitive-services
ms.subservice: face-api
ms.topic: how-to
ms.date: 03/01/2018
ms.author: sbowles
ms.devlang: csharp
ms.custom: devx-track-csharp
---

# Example: How to Analyze Videos in Real-time

This guide will demonstrate how to perform near-real-time analysis on frames taken from a live video stream. The basic components in such a system are:

- Acquire frames from a video source
- Select which frames to analyze
- Submit these frames to the API
- Consume each analysis result that is returned from the API call

These samples are written in C# and the code can be found on GitHub here: [https://github.com/Microsoft/Cognitive-Samples-VideoFrameAnalysis](https://github.com/Microsoft/Cognitive-Samples-VideoFrameAnalysis/).

## The Approach

There are multiple ways to solve the problem of running near-real-time analysis on video streams. We will start by outlining three approaches in increasing levels of sophistication.

### A Simple Approach

The simplest design for a near-real-time analysis system is an infinite loop, where each iteration grabs a frame, analyzes it, and then consumes the result:

```csharp
while (true)
{
    Frame f = GrabFrame();
    if (ShouldAnalyze(f))
    {
        AnalysisResult r = await Analyze(f);
        ConsumeResult(r);
    }
}
```

If our analysis consisted of a lightweight client-side algorithm, this approach would be suitable. However, when analysis happens in the cloud, the latency involved means that an API call might take several seconds. During this time, we are not capturing images, and our thread is essentially doing nothing. Our maximum frame-rate is limited by the latency of the API calls.

### Parallelizing API Calls

While a simple single-threaded loop makes sense for a lightweight client-side algorithm, it doesn't fit well with the latency involved in cloud API calls. The solution to this problem is to allow the long-running API calls to execute in parallel with the frame-grabbing. In C#, we could achieve this using Task-based parallelism, for example:

```csharp
while (true)
{
    Frame f = GrabFrame();
    if (ShouldAnalyze(f))
    {
        var t = Task.Run(async () => 
        {
            AnalysisResult r = await Analyze(f);
            ConsumeResult(r);
        }
    }
}
```

This code launches each analysis in a separate Task, which can run in the background while we continue grabbing new frames. With this method we avoid blocking the main thread while waiting for an API call to return, but we have lost some of the guarantees that the simple version provided. Multiple API calls might occur in parallel, and the results might get returned in the wrong order. This could also cause multiple threads to enter the ConsumeResult() function simultaneously, which could be dangerous, if the function is not thread-safe. Finally, this simple code does not keep track of the Tasks that get created, so exceptions will silently disappear. Therefore, the final step is to add a "consumer" thread that will track the analysis tasks, raise exceptions, kill long-running tasks, and ensure that the results get consumed in the correct order.

### A Producer-Consumer Design

In our final "producer-consumer" system, we have a producer thread that looks similar to our previous infinite loop. However, instead of consuming analysis results as soon as they are available, the producer simply puts the tasks into a queue to keep track of them.

```csharp
// Queue that will contain the API call tasks. 
var taskQueue = new BlockingCollection<Task<ResultWrapper>>();
     
// Producer thread. 
while (true)
{
    // Grab a frame. 
    Frame f = GrabFrame();
 
    // Decide whether to analyze the frame. 
    if (ShouldAnalyze(f))
    {
        // Start a task that will run in parallel with this thread. 
        var analysisTask = Task.Run(async () => 
        {
            // Put the frame, and the result/exception into a wrapper object.
            var output = new ResultWrapper(f);
            try
            {
                output.Analysis = await Analyze(f);
            }
            catch (Exception e)
            {
                output.Exception = e;
            }
            return output;
        }
        
        // Push the task onto the queue. 
        taskQueue.Add(analysisTask);
    }
}
```

We also have a consumer thread that takes tasks off the queue, waits for them to finish, and either displays the result or raises the exception that was thrown. By using the queue, we can guarantee that results get consumed one at a time, in the correct order, without limiting the maximum frame-rate of the system.

```csharp
// Consumer thread. 
while (true)
{
    // Get the oldest task. 
    Task<ResultWrapper> analysisTask = taskQueue.Take();
 
    // Await until the task is completed. 
    var output = await analysisTask;
     
    // Consume the exception or result. 
    if (output.Exception != null)
    {
        throw output.Exception;
    }
    else
    {
        ConsumeResult(output.Analysis);
    }
}
```

## Implementing the Solution

### Getting Started

To get your app up and running as quickly as possible, you will use a flexible implementation of the system described above. To access the code, go to [https://github.com/Microsoft/Cognitive-Samples-VideoFrameAnalysis](https://github.com/Microsoft/Cognitive-Samples-VideoFrameAnalysis).

The library contains the class FrameGrabber, which implements the producer-consumer system discussed above to process video frames from a webcam. The user can specify the exact form of the API call, and the class uses events to let the calling code know when a new frame is acquired or a new analysis result is available.

To illustrate some of the possibilities, there are two sample apps that use the library. The first is a simple console app, and a simplified version of it is reproduced below. It grabs frames from the default webcam, and submits them to the Face service for face detection.

:::code language="csharp" source="~/cognitive-services-quickstart-code/dotnet/Face/sdk/analyze.cs":::

The second sample app is a bit more interesting, and allows you to choose which API to call on the video frames. On the left-hand side, the app shows a preview of the live video, on the right-hand side it shows the most recent API result overlaid on the corresponding frame.

In most modes, there will be a visible delay between the live video on the left, and the visualized analysis on the right. This delay is the time taken to make the API call. One exception is the "EmotionsWithClientFaceDetect" mode, which performs face detection locally on the client computer using OpenCV, before submitting any images to Cognitive Services. This way, we can visualize the detected face immediately and then update the emotions once the API call returns. This is an example of a "hybrid" approach, where the client can perform some simple processing, and Cognitive Services APIs can augment this with more advanced analysis when necessary.

![HowToAnalyzeVideo](../../Video/Images/FramebyFrame.jpg)

### Integrating into your codebase

To get started with this sample, follow these steps:

1. Create an [Azure account](https://azure.microsoft.com/free/cognitive-services/). If you already have one, you can skip to the next step.
2. Create resources for Computer Vision and Face in the Azure portal to get your key and endpoint. Make sure to select the free tier (F0) during setup.
   - [Computer Vision](https://portal.azure.com/#create/Microsoft.CognitiveServicesComputerVision)
   - [Face](https://portal.azure.com/#create/Microsoft.CognitiveServicesFace)
   After the resources are deployed, click **Go to resource** to collect your key and endpoint for each resource. 
3. Clone the [Cognitive-Samples-VideoFrameAnalysis](https://github.com/Microsoft/Cognitive-Samples-VideoFrameAnalysis/) GitHub repo.
4. Open the sample in Visual Studio, and build and run the sample applications:
    - For BasicConsoleSample, the Face key is hard-coded directly in [BasicConsoleSample/Program.cs](https://github.com/Microsoft/Cognitive-Samples-VideoFrameAnalysis/blob/master/Windows/BasicConsoleSample/Program.cs).
    - For LiveCameraSample, the keys should be entered into the Settings pane of the app. They will be persisted across sessions as user data.
        

When you're ready to integrate, **reference the VideoFrameAnalyzer library from your own projects.** 

## Summary

In this guide, you learned how to run near-real-time analysis on live video streams using the Face, Computer Vision, and Emotion APIs, and how to use our sample code to get started.

Feel free to provide feedback and suggestions in the [GitHub repository](https://github.com/Microsoft/Cognitive-Samples-VideoFrameAnalysis/) or, for broader API feedback, on our [UserVoice](https://feedback.azure.com/d365community/forum/09041fae-0b25-ec11-b6e6-000d3a4f0858) site.

## Related Topics
- [Call the detect API](identity-detect-faces.md)
