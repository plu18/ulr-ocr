This demo is showing how to use OpenCV, tesseract and Python to extract **ultra low resolution (ULR) text** from images. In our case, the X-height is as small as 8 px without any anti-aliasing. The use case for this is system level testing of embedded devices.

[Tesseract](https://en.wikipedia.org/wiki/Tesseract_(software)) was originally written to do object character recognition (OCR) on high-resolution images like scanned pages and it does very well in this domain. But if you try to use it for low resolution text, like screen text, the first time, you are immediately frustrated. It simply does not work. And for ULR text, the outcome is even worse. The problem becomes more complicated by the fact that tesseract is not exactly trivial. First, there are dozens of configuration options which can only be understand by looking at the code and second, tesseract has some mechanism like *rejection*, which can lead to unexpected results.

This demo shows two things:
- How to use tesseract for ULR text.
- What to do if tesseract fails ... and in some cases it does. Using simpler techniques like *template matching* makes more sense then.

One of our input images looks like this. See the `doc/images` directory for another example.

![input image](doc/images/input_image.png)

When doing system level testing for embedded devices, we usually stimulate the device under test and check the result. The result in our case is what is displayed on the device's LCD screen (128x64 px).

Tesseract for ULR text
----------------------

As mentioned before, tesseract was written for scanned pages and these usually have very high resolution, say 300 dpi. That is why any attempt to pass ULR text images to tesseract is doomed to fail. It simply is not what tesseract is made for. In the best case, tesseract rejects all the text, because it considers it noise.

You should always keep in mind that tesseract does *not* pre-process images in any way, except for a thresholding step which converts the image to black/white. But, as a rule of thumb, OCR quality of the result is usually better by passing a black/white image to tesseract.

As a first pre-processing step, we scale the image by a factor of 6 using cubic interpolation. This boosts the resolution from around 50 dpi to 300 dpi and at the same time smooths the characters edges.

![input image scaled](doc/images/input_image_scaled.png)

The text looks still very stepped, but this is okay for now. We will use patterns and word lists later to fix errors from the character recognition step. Before we pass the image into tesseract, we apply binary thresholding. The threshold value is automatically computed by Otsu's algorithm which is available in OpenCV. Here is how the result looks like:

![input image after thresholding](doc/images/input_image_after_thresholding.png)

For us, the image looks less like "good" text, but tesseract internally processes black/white images only.

So, lets feed tesseract with out scaled and thresholded image. Without any additional options, we get this:

    $ tesseract doc/images/input_image_after_thresholding.png stdout
    05.11.2015
    Plausi.-check fa iled

    0.84

    %o

Well, not that bad, but it contains some errors which could really cause problems in automated testing. First, the date is incorrect. There is a `5` in the result instead of a `6`. The other errors are `%o` instead of the per-mille symbol and instead of `failed`, the two words `fa` and `iled` where recognized. Hmm.

Okay, to better understand what went wrong with the additional space, let's look at the intermediate results from tesseract. For this, create a file with name `config.txt` and insert the following lines:

    tessedit_rejection_debug 1
    debug_fix_space_level 2

This will enable debugging of the so-called rejection mechanism and the handling of spaces.

Run tesseract again and add the `config.txt` as last argument to the command line. Tesseract will output a lot of debug output, including the following:

	EXTRACTED (0): "failed/8 "
	non-dict or ambig word detected
	set_done(): done=0
	 : fa : R=20.74, C=-3.41959, F=1.1, Perm=8, xht=[29.1321,34.8369], ambig=1
	pos	NORM	NORM
	str	f	a
	state:	1 	3 
	C	-3.420	-2.371
	Permuter Type = 8
	Certainty: -3.419592     Rating: 20.739964
	Dict word: 8
	set_done(): done=0
	 : iled : R=47.3536, C=-5.67648, F=1.25, Perm=2, xht=[29.1321,33.9762], ambig=0
	pos	NORM	NORM	NORM	NORM
	str	i	l	e	d
	state:	1 	1 	1 	2 
	C	-1.608	-5.676	-1.990	-2.791
	Permuter Type = 2
	Certainty: -5.676478     Rating: 47.353603
	Dict word: 0
	TESTED (2): "fa/8 iled/2 "
	set_done(): done=0
	 : failed : R=60.5025, C=-5.67648, F=1.1, Perm=8, xht=[29.1321,33.9762], ambig=1
	pos	NORM	NORM	NORM	NORM	NORM	NORM
	str	f	a	i	l	e	d
	state:	1 	3 	1 	1 	1 	2 
	C	-3.420	-2.371	-1.608	-5.676	-1.990	-2.791
	Permuter Type = 8
	Certainty: -5.676478     Rating: 60.502533
	Dict word: 8
	TESTED (0): "failed/8 "
	RETURNED (2): "fa/8 iled/2 "

For now, the first and the last line are most interesting. From the first line, we can realize that tesseract was indeed able to recognize `failed`. But after some magic, we can see in the last line that tesseract decided not to choose `failed` as the result, but instead `fa iled`. That seem completely weird at first, but it is exactly how tesseract works internally. The actual character recognition is only one part of the text recognition story. After the character recognition, tesseract permutes the results to find better matches in case the character recognition worked poorly. And this is what happened here. The result from character recognition is `failed`, but `fa iled` had a better score in the end. So, how to tell tesseract what we want?

We have two options. The first option is to play around with tesseract's parameters for "fixing" spaces. But that requires a lot of knowledge about tesseract's internals. The second one is to provide a so-called user dictionary and to disable some internal choices like in the `fa iled` case. The number 8 in `fa/8 iled/2` indicates that `fa` was chosen because it is in the so-called system dictionary. By disabling it and providing an explicit list of expected words - what is usually possible for embedded devices - we can make the choices made by tesseract much more predictable.

Create a file called `words.txt` and add all the expected words:

    Plausi.
    check
    failed

Additionally, add the following lines to your `config.txt`:

    load_freq_dawg 0
    load_system_dawg 0

Call tesseract again with the word list.

    $ tesseract doc/images/input_image_after_thresholding.png stdout --user-words words.txt config.txt

In the debug output, you will still find `fa/2 iled/2`, but it is now rejected because neither of the words is in the system dictionary anymore. The OCR result is:

    05.11.2015
    Plausi.-check failed

    0.84

    %o

Using a word list is optional and in this case `failed` is still detected, but having a word list makes the result more predictable.

Two errors remain. The incorrect date and the incorrect unit character for per-mille. In both cases I decided both not to push tesseract any further. For the per-mille symbol, I am not even sure if tesseract is trained to recognize it. It seems more reasonable to either substitute `%o` with `‰` in the result or to use another approach, e.g. template matching. The latter is exactly what we do below. The problem with the `5` instead of the `6` might be caused by the poor quality of the input character which has a height of only 5 px. We simply do not use it, but it would be another candidate for template matching.


Template matching for ULR text
------------------------------

From the [OpenCV documentation](https://docs.opencv.org/2.4/doc/tutorials/imgproc/histograms/template_matching/template_matching.html):

>  Template matching is a technique for finding areas of an image that match (are similar) to a template image (patch).

The nice thing about template matching is its simplicity. In our system testing context, it is a good choice to detect icons or alike. Here, we exercise it to "recognize" individual characters like our `‰` symbol or the sequence `µg/L`.

To apply template matching, we have to create the patches as individual image files. In our case, we have two of them. One for the `‰` and another one for the `µg/L`:

![ug_L](templates/units/ug_L.png)
![per-mille](templates/units/per-mille.png)

Using these patterns is then just a matter of a few OpenCV calls. One to compute the "similarity" with the pattern and another one to find a minimum. See the source code for details.


Installation
------------

First, install tesseract version 3.4 or higher. Then, install the necessary Python
dependencies using `pip`.

    sudo pip3 install -r requirements.txt

Run
---

Run the code on the images like this:

    python3 run.py new_highscore.png