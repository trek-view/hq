---
date: 2022-05-13
title: "Creating a Video File Ready to be Uploaded to the Google Street View API"
description: "Using ffmpeg to create a video file with accompanying GPS telemetry."
categories: developers
tags: [Google, Street View, video, ffmpeg]
author_staff_member: dgreenwood
image: /assets/images/blog/2022-05-13/streetview-rawgpstimeline-meta.jpg
featured_image: /assets/images/blog/2022-05-13/streetview-rawgpstimeline-sm.jpg
layout: post
published: true
redirect_from:
  - /blog/2022/create-google-street-view-video-publish-api
---

**Using ffmpeg to create a video file with accompanying GPS telemetry.**

[A few years ago I talked about creating a 360 video from frames](/blog/turn-360-photos-into-360-video).

For example, [when you want to upload them to the Street View Publish API](/blog/upload-video-street-view-publish-api).

In that last post I described how to upload the video, but only briefly touched on how to create the video. Well, a year late, here is a how-to post explaining it.

## Packing the video

Creating the video is fairly trivial, but you should keep a few things in mind.

[Google Street View Ready cameras](https://www.google.com/streetview/contacts-tools/) generate videos at a framerate between 5 and 7 frames-per-second. I think part of this is due to battery efficiency when shooting, but I am not sure if it is also for another reason (e.g. Google server prefers a slower framerate). As I could not find any docs on the subject I will blindly follow what other camera manufacturers do and use 5 FPS as the output video frame rate.

[According to many ad-hoc observations in various Google Street View groups](https://www.facebook.com/groups/366117726774216), Street View servers also appear to prefer shorter segments, often rejecting longer videos. Again, this is undocumented but I will play it safe and pack videos with a length not exceeding 60 seconds. 

This means at a frame rate of 5 FPS (0.2 seconds per frame), each video will contain a maximum of 300 frames (60 * 5).

So first thing, batch your extracted photos ([extract from a video first, if needed](/blog/turn-360-video-into-timelapse-images-part-1))) into groups of 300.

Then pack into a video like so;

```shell
ffmpeg -r 5 -i PHOTO_%06d.jpg -c:v libx264 -pix_fmt yuv420p OUTPUT.mp4
```

I won't add any global metadata to the video yet, here's why...

## Adjusting the GPS

[As shown in the `gopro_fusion_timelapse_uploader.py` script I referenced here](/blog/upload-video-street-view-publish-api),  positional information for the video can be passed separately (as a `rawGPSTimeline` object) alongside the video when uploading to Street View (as opposed to writing full telemetry into the video in [CAMM](https://developers.google.com/streetview/publish/camm-spec) or [GPMD](https://github.com/gopro/gpmf-parser) format).

By setting the framerate at a fixed 5 FPS we totally ignore the actual time spacing between photos, but that doesn't matter.

That's fine. The real time spacing between photos does not matter when it comes to Street View (e.g. Google does not really care if photo was taken at the time reported).

However, it does matter the time spacing between GPS points matches the time spacing between the frames in the video. This we do need to adjust for.

I talked a bit about this problem with GoPro's [TimeWarp](/blog/turn-gopro-timewarp-video-into-timelapse-images) and [TimeLapse](/blog/turn-gopro-timelapse-video-into-timelapse-images) video modes previously.

Like these modes, what we have from the above command is usually a photo sped up from real-time. So like these modes, we need to modify the GPS times for the photos.

To do this, we can first extract all the `SubSecDateTimeOriginal`, `GPSLatitude`, `GPSLongitude`, and `GPSAltitude` values into a simple text file with the format:

`SubSecDateTimeOriginal`,`GPSLatitude`,`GPSLongitude`,`GPSAltitude`

The first `SubSecDateTimeOriginal` value (for the first frame in the video) will remain unchanged.

We know that the second frame is 0.2 seconds later in the video, so the second frame `SubSecDateTimeOriginal` needs to be adjusted by adding 0.2 seconds to the first frames `SubSecDateTimeOriginal` value: `2020:04:13 15:37:22.444` + 0.2 = `2020:04:13 15:37:22.644`. 

The 3rd frame time will be photo 2 + 0.2 (`2020:04:13 15:37:22.844`), and so on.

## Creating a `rawGpsTimeline`

Once the timestamps have been updated to match the video frame rate, we now need to convert to a format Street View expects.

[In the `gopro_fusion_timelapse_uploader.py` script, a `rawGpsTimeline` object is created that looks like this](https://github.com/smarquardt/samples-for-svpub/blob/master/video_upload/gopro_fusion_timelapse_uploader.py#L309):


```json
"rawGpsTimeline": [
	{
		"latLngPair": {	
			"latitude": lat,
			"longitude": lon
		},
		"altitude": alt,
		"gpsRecordTimestampUnixEpoch": {
			"seconds": epoch
		},
		"gpsSource": source
	}
]
```

For example;

```json
"rawGpsTimeline": [
  {
    "latLngPair": {
      "latitude": 90,
      "longitude": 90
    },
    "altitude": 90,
    "gpsRecordTimestampUnixEpoch": "2014-10-02T15:01:23.045123456Z",
  },
  {
    "latLngPair": {
      "latitude": 80,
      "longitude": 80
    },
    "altitude": 80,
    "gpsRecordTimestampUnixEpoch": "2014-10-02T15:01:24.045123456Z"
  }
]
```

Remember, if you create videos in batches of 300 frames, you need to create a `rawGpsTimeline` for each video.

## Add metadata to video (optional)

Finally, we can add some global metadata to the video(s) to make them easier to work with in the future (though this step is entirely optional, and not required to upload the video to Street View).

This can be done by copying the metadata from the first frame, into the video file like so:

```shell
exiftool -TagsFromFile FIRSTFRAME.jpg "-all:all>all:all" OUTPUT.mp4
```

`FIRSTFRAME.jpg` should be substituted for the first photo frame in the video. If you've got multiple videos (as a result of limiting video frames), you will need to find the first frame for each video. In my case I batched videos into 300 frames, so the first frame for the first video will be the 1st frame in sequence, then for the second video the 301st frame in sequence, third video 601st frame, etc.

In the case of 360 (equirectangular) videos you'll also need to use Google's Spatial Media Metadata Injector to add the required 360 metadata (photos use the `XMP-GPano:ProjectionType` tag, whilst videos use the `XMP-GSpherical:ProjectionType` tag). [A demo of how to use the Spatial Media Metadata Injector is covered in this post](/blog/introduction-to-xmp-namespaces).

## Upload the file

[Described previously in this post](/blog/upload-video-street-view-publish-api).