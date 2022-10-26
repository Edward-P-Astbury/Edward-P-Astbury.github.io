---
published: true
---
## Creating Facial Recognition using Emgu CV within a Windows Form

This is a setup guide in how to get [facial recognition](https://en.wikipedia.org/wiki/Facial_recognition_system) working within a Windows Form project.

### Setting up the project

1. Create a new Windows Form (.NET Framework) project

![Windows Form new project]({{site.baseurl}}/images/windows-form-new-project.png)

2. Add the following [Emgu CV](https://www.nuget.org/packages/EmguCV) NuGet package to the project. 

3. Run a initial build of the project to ensure bin and debug directories are created

4. in your project's `WindowsFormsFaceRecognition\WindowsFormsFaceRecognition\bin\Debug` directory create a new folder named `Haarcascade`. 

5. The following [XML](https://github.com/opencv/opencv/blob/master/data/haarcascades/haarcascade_frontalface_alt.xml) file containing serialized Haar Cascade detector of faces needs to be added to the newly created `Haarcascade` directory. Harr Cascade is one of the more powerful face detection algorithms.

### Coding the project

The full source code for the project can be found [here](https://github.com/Edward-P-Astbury/WindowsFormsFaceRecognition.git)

All the code for the facial recognition resides within the [FaceRecognition.cs](https://github.com/Edward-P-Astbury/WindowsFormsFaceRecognition/blob/f6a31106cbc2a1a8cf114713e8dfa58603e0f266/WindowsFormsFaceRecognition/FaceRecognition.cs) and the [Form1.cs](https://github.com/Edward-P-Astbury/WindowsFormsFaceRecognition/blob/f6a31106cbc2a1a8cf114713e8dfa58603e0f266/WindowsFormsFaceRecognition/Form1.cs) classes

The _important_ methods pertaining to the facial recognition are as follows:

**DetectFace()**

```csharp
        private void DetectFace()
        {
            Image<Bgr, byte> image = frame.Convert<Bgr, byte>();
            Mat mat = new Mat();
            CvInvoke.CvtColor(frame, mat, ColorConversion.Bgr2Gray);
            CvInvoke.EqualizeHist(mat, mat);
            Rectangle[] array = CascadeClassifier.DetectMultiScale(mat, 1.1, 4);
            if (array.Length != 0)
            {
                foreach (Rectangle rectangle in array)
                {
                    CvInvoke.Rectangle(frame, rectangle, new Bgr(Color.LimeGreen).MCvScalar, 2);
                    SaveImage(rectangle);
                    image.ROI = rectangle;
                    TrainedImage();
                    CheckName(image, rectangle);
                }
            }
            else
            {
                personName = "";
            }
        }
```

The `DetectFace()` method works by getting an Emgu.CV `Image` of the current frame and saving the output to a [`Mat`](https://docs.opencv.org/4.x/d3/d63/classcv_1_1Mat.html) object (used to store complicated vectors, matrices, grayscale/colour images and so forth). The colour space is also converted from BGR to GRAY. In simple terms, the image is converted to a grayscale image as grayscale images require less complicated algorithms, ultimately reducing computational overhead.

Once the image is converted to a grayscale image the `CVInvoke.EqualizeHist(mat, mat)` function is called which takes in our `mat` object and then makes the algorithm normalises the brightness and increases the contrast of the image.

We then create a array of System.Drawing `Rectangle` objects and assign it to `CascadeClassifier.DetectMultiScale(mat, 1.1, 4)` which returns rectangular regions in a image that the cascade has been trained for. In this case the cascade has been provided a frontal face training data set provided by the following [XML](https://github.com/opencv/opencv/blob/master/data/haarcascades/haarcascade_frontalface_alt.xml).

The next part of the function is relatively self-explanatory. We iterate through our `Rectangle` array, draw the visual rectangle with `CvInvoke.Rectangle()`, save the image, set the image ROI (region of interest) to the rectangle capturing the face, and call the `TrainedImage()` and `CheckName(image, rectangle)` methods.

***TrainedImage()***

```csharp
        private void TrainedImage()
        {
            try
            {
                int number = 0;
                trainedFaces.Clear();
                personLabels.Clear();
                names.Clear();
                string[] files = Directory.GetFiles(Directory.GetCurrentDirectory() + "\\Image", "*.jpg", SearchOption.AllDirectories);

                foreach (string text in files)
                {
                    Image<Gray, byte> item = new Image<Gray, byte>(text);
                    trainedFaces.Add(item);
                    personLabels.Add(number);
                    names.Add(text);
                    number++;
                }

                eigenFaceRecognizer = new EigenFaceRecognizer(number, distance);
                eigenFaceRecognizer.Train(trainedFaces.ToArray(), personLabels.ToArray());
            }
            catch
            {
                // exception handling
            }
        }
```

The `TrainedImage()` method operates by iterating through our saved images and then utilising the `EigenFaceRecognizer` class - facial recognition class that works with grayscale images. The `eigenFaceRecognizer.Train` method is called which as the name of the method suggests it trains the face recogniser to be trained with the passed in images and labels.

***CheckName(Image<Bgr, byte>, Rectangle)***

```csharp
        private void CheckName(Image<Bgr, byte> resultImage, Rectangle face)
        {
            try
            {
                if (FaceDetectionOn)
                {
                    Image<Gray, byte> image = resultImage.Convert<Gray, byte>().Resize(100, 100, Inter.Cubic);
                    CvInvoke.EqualizeHist(image, image);
                    FaceRecognizer.PredictionResult predictionResult = eigenFaceRecognizer.Predict(image);
                  
                    if (predictionResult.Label != -1 && predictionResult.Distance < distance)
                    {
                        picBoxFrameSmall.Image = trainedFaces[predictionResult.Label].Bitmap;
                        personName = names[predictionResult.Label].Replace(Environment.CurrentDirectory + "\\Image\\", "").Replace(".jpg", "");
                        CvInvoke.PutText(frame, personName, new Point(face.X - 2, face.Y - 2), FontFace.HersheyPlain, 1.0, new Bgr(Color.LimeGreen).MCvScalar);
                    }
                    else
                    {
                        CvInvoke.PutText(frame, "UNKNOWN", new Point(face.X - 2, face.Y - 2), FontFace.HersheyPlain, 1.0, new Bgr(Color.OrangeRed).MCvScalar);
                    }
                }
            }
            catch
            {
                // error handling
            }
        }
    }
```

The `CheckName(Image<Bgr, byte>, Rectangle)` method funtions by passing in a `Image<Bgr, byte>` which will then be assigned to a local `Image<Gray, byte>` variable through calling the `.Convert()` method which enables the conversion of a image to a specified colour and depth. The image is also resized at this stage using `.Resize()` to align the image with the same size of the .JPG images.

As done before we utlise the `CVInvoke.EqualizeHist(image, image)` to normalise the image attributes - brightness and contrast.

Importantly we utilise the `.Predict()` method which takes in a image and predicts the label attribute of the image based on the previously provided training images to the `eigenFaceRecognizer` object. If a face is detected, we will successfully pass the `if` check and display the image of the individual through a Windows Form `PictureBox`. Then we assign the person's name to the face based on the `predictionResult` Label field. After this we simply place some text of the person's name utilising `CvInvoke.PutText`.

### Project Demonstration

The below image shows the final result of the program.

![program-demo-image]({{site.baseurl}}/images/program-demo.png)

The three buttons located on the left-hand plane are responsible for all the core program features.

- Video Camera Capture - opens the currently connected webcam
- Save Image - saves a frame from the camera stream and trims the image region to the individual face
- Scan Faces - Enables the face detection based on the saved images

After entering a name into the text-box field and saving the image via the `Save Image` button you are able to use the `Scan Faces` button and see how the facial recognition is now able to determine who you are!

![edward-face-demo]({{site.baseurl}}/images/demo-edward-face.png)

### Conclusion

It is quite clear that through utilising a pre-existing .NET wrapper library to the open source computer vision library, OpenCV, we are able to achieve facial recognition with minimal code as a lot of the complexity is abstracted away from us due to leveraging the [Emgu CV](https://www.nuget.org/packages/EmguCV) library. This in tandem with the already optimised XML [data set](https://github.com/opencv/opencv/blob/master/data/haarcascades/haarcascade_frontalface_alt.xml) we are left with only needing to invoke/call the necessary functions provided by Emgu CV.

Unfortunately in my development I was unable to implement the [Azure Face API](https://azure.microsoft.com/en-us/products/cognitive-services/face/#overview). This was primarily due to the restrictions of certain features I wanted to employ. The detection of face attributes, such as, smile, hair colour, facial hair, etc. is currently being limited to Microsoft managed customers and partners due to the responsible AI principles. 