---
title: "Downloading Mercari Listing Images"
date: 2021-02-05T21:23:43-06:00
---

Yet another marketplace, [Mercari](https://www.mercari.com/) is rather mediocre with several rough edges. As a ~~digital hoarder~~ amateur archivist, downloading and archiving images from Mercari listings is rather difficult. Their frontend uses cropped, low resolution images and seems to enjoy making image viewing a very painful experience by hijacking your cursor and scroll wheel. Although I keep trying, a right click and "Save As..." has yet to save me.

Rather than continuing to fight a website and lose, the easiest solution is to write a simple shell script for downloading the product images for a given Mercari listing.

## Mercari's Dynamic Image Generation
To make an example of a random Mercari listing, consider this url: `https://www.mercari.com/us/item/m51613244349/`

`m51613244349` is the Mercari listing's uniquely assigned id. Currently it seems to use 11-digit values (excluding `m`), but a more flexible regex will likely pay off at some point in the future: `(m)[0-9]{8,16}`

Opening Firefox's [Network Monitor](https://developer.mozilla.org/en-US/docs/Tools/Network_Monitor), you should see several requests to `mercari-images.global.ssl.fastly.net` (depending on the amount of images in the product listing). Each of these requests is used to load a small preview thumbnail of a larger image.

For example:

`m51613244349_1.jpg?1612581994&w=200&h=200&fitcrop&sharpen`
`m51613244349_2.jpg?1612581994&w=200&h=200&fitcrop&sharpen`
`m51613244349_3.jpg?1612581994&w=200&h=200&fitcrop&sharpen`

Our favorite Mercari listing id starts the filepath, proceeded by an incrementing number that we'll creatively call the *file number*. The file number starts at 1 and ends at a maximum value of 12 (although this is an arbitrary limit imposed by Mercari).

Next up is an epoch timestamp value in seconds. It is likely used for cache invalidation purposes. You may omit the value seemingly without issue. 

Finally, our `w`, `h`, `fitcrop` and `sharpen` parameters hint that this endpoint is likely doing dynamic image manipulation. This allows the Mercari frontend to request whatever it likes without worrying whether or not the image has been previously generated.

But why should frontend developers have all the fun? We can use this endpoint ourselves to download uncropped, unresized product listing images.

Taking a previous request, `m51613244349_1.jpg?1612581994&w=200&h=200&fitcrop&sharpen`, let's rebuild it into something more useful. I don't want cropped or filtered images, and cache timestamps be damned.

Providing a height and width means we'll enforce a specific aspect ratio and could distort the image. Luckily their endpoint doesn't seem to mind omitting either of the values, ensuring it returns the image in its original aspect ratio.

While we're at it, let's bump that 200px height to something... bigger.

Finally we land at `m51613244349_1.jpg?h=8192`

## Looping It

Given the file number has rigid bounds and is consistently incrementing, we can write a basic shell script to increment a variable and wget whatever image exists at that url, with whatever request parameters we like.

```bash
#!/bin/sh

MERCARI_LISTING=$1
IMG_SERVER_INDEX=1
IMG_HEIGHT=8192

while true
do
    URL="https://mercari-images.global.ssl.fastly.net/photos/${MERCARI_LISTING}_${IMG_SERVER_INDEX}.jpg?h=${IMG_HEIGHT}"
    FILENAME="${MERCARI_LISTING}_${IMG_SERVER_INDEX}.jpg"
    IMG_SERVER_INDEX=$((IMG_SERVER_INDEX+1))   
    wget --no-verbose --output-document="$FILENAME" "$URL"
    if [ $? -ne 0 ]
    then
        printf '\xE2\x9C\x85\x0A'
        break
    fi
done
```

Instead of incrementing through file numbers 1 to 12, which could arbitrarily change in the future, I've opted to read the HTTP status code returned by wget. Fastly seems to return a [403 Forbidden](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/403) status when the url doesn't exist (or when they don't want us to know it does...). You can test this behavior if you use a file number of 0, 13 or anything higher than the total number of images in the product listing.

## But wait!

*Why not scrape the page (spoof an endpoint call, etc) to know how many images there are?*

That requires the same amount of requests as iterating until you hit the end (n+1) and only adds extra complexity. Plus, the Mercari frontend seems to be a mess of dynamically named (or obfuscated) elements and Javascript.

*How large of an image can you request?*

I don't know. Requesting a height of something like 2048px ensures you're likely getting a decent quality when possible, without trying to hack together a "4K" image out of something taken on a iPhone 3GS.

*What if they shut it down?*

I wouldn't blame them. Allowing the frontend to dynamically generate images on the fly seems like a mess for caching purposes, page load speeds, and backend resources. If they do, I hope they provide someway to download the original images for archival and reference purposes.
