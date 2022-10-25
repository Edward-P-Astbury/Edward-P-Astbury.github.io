---
published: false
---
## Creating Facial Recognition using Emgu CV within a Windows Form

This is a setup guide in how to get [facial recognition](https://en.wikipedia.org/wiki/Facial_recognition_system) working within a Windows Form project.

### Setting up the project

1. Create a new Windows Form project

![Windows Form new project]({{site.baseurl}}/_posts/windows-form-new-project.png)

2. Add the following [Emgu CV](https://www.nuget.org/packages/EmguCV) NuGet package to the project. 

3. Run a initial build of the project to ensure bin and debug directories are created

4. in your project's `WindowsFormsFaceRecognition\WindowsFormsFaceRecognition\bin\Debug` directory create a new folder named `Haarcascade`. 

5. The following [XML](https://github.com/opencv/opencv/blob/master/data/haarcascades/haarcascade_frontalface_alt.xml) file containing serialized Haar cascade detector of faces needs to be added to the newly created `Haarcascade` directory

### Coding the project

The full source code for the project can be found [here](https://github.com/Edward-P-Astbury/WindowsFormsFaceRecognition.git)

All the code for the facial recognition resides within the [FaceRecognition.cs](https://github.com/Edward-P-Astbury/WindowsFormsFaceRecognition/blob/f6a31106cbc2a1a8cf114713e8dfa58603e0f266/WindowsFormsFaceRecognition/FaceRecognition.cs) and the [Form1.cs](https://github.com/Edward-P-Astbury/WindowsFormsFaceRecognition/blob/f6a31106cbc2a1a8cf114713e8dfa58603e0f266/WindowsFormsFaceRecognition/Form1.cs) classes

The important methods pertaining to the facial recognition are as follows:

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

The `DetectFace()` method works by getting an Emgu.CV `Image` of the current frame and saving the output destination to a [`Mat`](https://docs.opencv.org/4.x/d3/d63/classcv_1_1Mat.html) object (used to store complicated vectors, matrices, grayscale/colour images and so forth). The colour space is also converted from BGR to GRAY. In simple terms, the image is converted to a grayscale image as grayscale images require a less complicated algorithm, ultimately reducing computational overhead.

After the image is converted to a gray the `CVInvoke.EqualizeHist(mat, mat)` function is called which takes in our `mat` object and normalises the brightness and increases the contrast of the image.

