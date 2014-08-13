libamivideo
===========
Commodore Amiga hardware uses a different means of storing and displaying
graphics compared to "modern hardware", such as PCs.

Pixels
------
Historically, the most widely used display mode for games and many multimedia
applications on PCs is a 320x200 resolution screen with a 256 color palette. In
this display mode, pixels are encoded as bytes in which each byte value refers
to an index of the palette.

These days, PCs have much more system resources and can directly use any possible
color value for a pixel. The most common way to support this by encoding a pixel
as 4 bytes in which three byte refers to the red, blue and green intensity
values. One byte is used for padding.

Amiga hardware uses a completely different approach -- pixels are encoded as
bitplanes rather than bytes.

When using bitplane encoding, an image is stored multiple times in memory. In
every image occurence a bit represents a pixel. In the first occurence, a bit is
the least significant bit of an index a value. In the last occurence a bit is the
most significant bit of an index value. By adding all the bits together we can
determine the index value of the palette to determine a pixel's color value.

Palette
-------
Color values on PCs are stored in 4-byte color registers, in which one byte
is used for the red color intensity, one for green color intensity and one
for blue color intensity. One byte is used for padding.

The Amiga display have a palette with predefined color values that are stored in
either 32 color registers (`OCS` and `ECS` chipset) containing 12-bit color
values or 256 color registers (`AGA` chipset) storing 32-bit color values.

Furthermore, the Amiga video chips have special screen modes to squeeze more
colors out of the amount of color registers available. The Extra Half Brite (EHB)
screen mode is capable of displaying 64 colors out of a predefined 32, in which
the last 32 colors values have half of the intensity of the first 32 colors.

The Hold-and-Modify (HAM) screen mode is used to modify a color component of the
previous adjacent pixel or to provide a new color from the given palette. This
screen mode makes it possible to display all possible color values with some
loss of quality.

Resolutions
-----------
In addition to colors, resolutions are also different on the Amiga compared to
modern platforms.

On PCs, resolutions refer to the amount of pixels per scanline and the amount
of scanlines per screen.

On the Amiga, resolutions only refer to the amount of pixels per scanline and
only a few fixed resolutions can be used.

For example, a high resolution screen has twice the amount of pixels per scanline
compared to a low resolution screen. A super hires screen has double the amount
of pixels per scanline compared to a high resolution screen. Moreover, a low
resolution pixel is twice a wide as a high resolution pixel and so on.

Vertically, there are only two options. In principle, there are a fixed amount of
scanlines on a display. The amount of scanlines can be doubled, using a so-called
_interlace_ screen mode. However, interlace screen modes have a big drawback --
they draw the odd scanlines in one frame and the even ones in another. On
displays with a short after glow, flickering may be observed.

If we would convert an Amiga image directly to a PC display, we may observe odd
results in some cases. For example, a non-interlaced high resolution image looks
twice as wide on a PC display than on an Amiga display. To give it the same look,
we must correct its aspect ratio by doubling the amount of scanlines on the PC
display.

Features
========
The purpose of this library is to cope with the differences of PCs and Amiga
displays in an easy way -- it acts as a conversion library for the Amiga video
chips to modern hardware and vice versa. More specifically, this library offers
the following features:

* Converting a 12/24-bit color palette to a 32-bit true color palette and vice versa
* Emulation of the Extra Half-brite (EHB) and Hold-and-Modify (HAM) screen modes
* Converting bitplanes to chunky pixels and vice versa
* Converting bitplanes to true color pixels
* Correcting the aspect ratio of Amiga images

In principle, this library does not do any allocation of source or destination
surfaces that contain graphics data, but acts as an _adapter_ that is placed in
the middle of these two instances that are already created by other means.

The source and destination services can be nearly anything, such as something
generated by a library or a real Amiga viewport. The only requirement is that
they must store pixel data in their "raw" format that the functions of this
library can access.

Converting planar graphics to chunky/RGB graphics
=================================================
This section convers a simple example in which we convert an Amiga viewport
surface to something that can be displayed on a PC.

Acquiring a planar graphics source
----------------------------------
We use the following imaginary struct as example representing an Amiga viewport:

    typedef struct Color
    {
        UBYTE r, g, b;
    }
    Color;
    
    struct
    {
        ULONG width;
        ULONG height;
        UWORD bitplaneDepth;
        ULONG viewportMode;
        
        Color color[32];
        UBYTE *bitplanes[6];
    }
    viewport;

The above struct represents an Amiga viewport having a specific width
(in pixels), height (in scanlines), a bitplane depth (corresponding to the amount
of colors used), a viewport mode value that contains Amiga specific display
mode bits, such as extra half-brite mode, and a palette consisting of 32 colors.

The last member of the struct is an array containing bitplane encoded pixels.

Creating a conversion struct instance
-------------------------------------
To be able to convert the data in the viewport, we must create an instance of the
`amiVideo_Screen` struct that acts as an adapter for the conversion processes.

In the following code fragment, we configure a screen instance adopting the
dimensions, display settings, and palette of the example viewport struct shown
earlier:

    #include <libamivideo/screen.h>
    
    /* Initialise screen with screen settings */
    amiVideo_Screen screen;
    
    /*
     * Create a screen conversion struct instance with the viewport's width,
     * height, bitplane depth and viewport mode. We use 4-bits per color
     * component to simulate OCS/ECS display modes.
     */
    amiVideo_initScreen(&screen, viewport->width, viewport->height, viewport->bitplaneDepth, 4, viewport->viewportMode);
    
    /* Set the bitplane palette to the viewport's palette */
    amiVideo_setBitplanePaletteColors(&screen->palette, color, 32);
    
    /* Set the bitplane pointers to the pointers in the viewport */
    amiVideo_setScreenBitplanePointers(&screen, viewport->bitplanes);

After having configured the screen adapter, we can use it to convert the viewport
to something that can be displayed on modern hardware, with or without correcting
its aspect ratio. In the next sections, we explain how this can be done.

Allocating a target surface and performing the conversion
---------------------------------------------------------
In the examples used in the following sub sections, we use the [SDL](http://www.libsdl.org)
library to allocate a target pixel surface storing pixels in the converted
format.

There are various output formats supported. Some of them are more memory
efficient and have certain restrictions, others give a better equivalent pixel
experience on modern platforms. Moreover, we show how the actual conversion
process can be executed.

Converting to uncorrected chunky pixels format
----------------------------------------------
The simplest output format is uncorrected chunky pixels in which each output
pixel is a byte referring to an index of the palette. Moreover, color components
of the palette are converted from 4 bits to 8 bits:
    
    #include <SDL.h>
    
    /* Create a SDL surface having 8 bits per pixel in which the output is stored */
    SDL_Surface *surface = SDL_CreateRGBSurface(0, screen.width, screen.height, 8, 0, 0, 0, 0);
    
    /*
     * Convert the colors of the palette from 4 bits per color component to 8
     * bits per color component.
     */
    amiVideo_convertBitplaneColorsToChunkyFormat(&screen.palette);
    
    /* Set the palette of the target SDL surface */
    if(SDL_SetPaletteColors(surface->format->palette, (SDL_Color*)screen.palette.chunkyFormat.color, 0, screen.palette.chunkyFormat.numOfColors) == 0)
    {
        fprintf(stderr, "Cannot set palette of the surface!\n");
        return 1;
    }
    
    /* Sets the uncorrected chunky pixels pointer of the conversion struct to that of the SDL pixel surface */
    amiVideo_setScreenUncorrectedChunkyPixelsPointer(&screen, surface->pixels, surface->pitch);
    
    /* Convert the bitplanes to chunky pixels */
    
    if(SDL_MUSTLOCK(surface) && SDL_LockSurface(surface) != 0)
    {
        fprintf(stderr, "Cannot lock the surface!\n");
        return 1;
    }
    
    amiVideo_convertScreenBitplanesToChunkyPixels(&screen);
    
    if(SDL_MUSTLOCK(surface))
        SDL_UnlockSurface(surface);

Converting to uncorrected RGB pixels format
-------------------------------------------
Chunky graphics output format is memory efficient and suffices for nearly all
Amiga screen modes, except for the HAM screenmodes which support more than 256
colors due to a compression technique and 24 and 32 bitplanes modes which are
used by so-called deep ILBM images.

To convert to RGB in which every 4 bytes refer to a pixel's color value, we can
do the following:

    #include <SDL.h>
    
    /* Create a SDL surface having 24 bits per pixel in which the output is stored */
    SDL_Surface *surface = SDL_CreateRGBSurface(0, screen.width, screen.height, 24, 0, 0, 0, 0);
    
    /* Set the uncorrected RGB pixels pointer of the conversion struct to that of the SDL pixel surface */
    amiVideo_setScreenUncorrectedRGBPixelsPointer(&screen, surface->pixels, surface->pitch, TRUE, surface->format->Rshift, surface->format->Gshift, surface->format->Bshift, surface->format->Ashift);
    
    /* Convert the bitplanes to RGB pixels */
    if(SDL_MUSTLOCK(surface) && SDL_LockSurface(surface) != 0)
    {
        fprintf(stderr, "Cannot lock the surface!\n");
        return 1;
    }
    
    amiVideo_convertScreenBitplanesToRGBPixels(&screen);
    
    if(SDL_MUSTLOCK(surface))
        SDL_UnlockSurface(surface);

Converting to corrected chunky pixels format
--------------------------------------------
For the previous output formats, every output pixel corresponds to a pixel in the
original input format. However, as explained earlier, because of the different
way pixels and scanlines are divided, we may observe odd results that we need to
correct, for example by doubling the pixels or scanlines.

The following example code converts a planar screen to a _corrected_ chunky
surface:
    
    #include <SDL.h>
    
    SDL_Surface *surface;
    
    /*
     * We set the size of a lowres pixel. A lowres pixel has the same size of
     * two hires pixels. 2 will typically suffice for most images. To support
     * super hires displays this value needs to be set to 4. However, this
     * also results in a much bigger output picture.
     */
    unsigned int lowresPixelScaleFactor = 2;
    amiVideo_setLowresPixelScaleFactor(&screen, lowresPixelScaleFactor);
    
    /*
     * Create a SDL surface having 8 bits per pixel in which the output is stored.
     * The correctedFormat sub struct provides us the dimensions of the corrected
     * surface.
     */
    surface = SDL_CreateRGBSurface(0, screen.correctedFormat.width, screen.correctedFormat.height, 8, 0, 0, 0, 0);
    
    /*
     * Convert the colors of the palette from 4 bits per color component to 8
     * bits per color component.
     */
    amiVideo_convertBitplaneColorsToChunkyFormat(&screen.palette);
    
    /* Set the palette of the target SDL surface */
    if(SDL_SetPaletteColors(surface->format->palette, (SDL_Color*)screen.palette.chunkyFormat.color, 0, screen.palette.chunkyFormat.numOfColors) == 0)
    {
        fprintf(stderr, "Cannot set palette of the surface!\n");
        return 1;
    }
    
    /* Set the corrected chunky pixels pointer of the conversion struct to the SDL pixel surface */
    amiVideo_setScreenCorrectedPixelsPointer(&screen, surface->pixels, surface->pitch, 1, TRUE, 0, 0, 0);
    
    /* Convert the bitplanes to corrected chunky pixels */
    if(SDL_MUSTLOCK(surface) && SDL_LockSurface(surface) != 0)
    {
        fprintf(stderr, "Cannot lock the surface!\n");
        return 1;
    }
    
    amiVideo_convertScreenBitplanesToCorrectedChunkyPixels(&screen);
    
    if(SDL_MUSTLOCK(surface))
        SDL_UnlockSurface(surface);

Converting to corrected RGB pixels format
-----------------------------------------
The following example code converts a planar screen to a corrected RGB surface:
    
    #include <SDL.h>
    
    SDL_Surface *surface;
    
    /*
     * We set the size of a lowres pixel. A lowres pixel has the same size of
     * two hires pixels. 2 will typically suffice for most images. To support
     * super hires displays this value needs to be set to 4. However, this
     * also results in a much bigger output picture.
     */
    unsigned int lowresPixelScaleFactor = 2;
    amiVideo_setLowresPixelScaleFactor(&screen, lowresPixelScaleFactor);
    
    /*
     * Create a SDL surface having 24 bits per pixel in which the output is
     * stored. The correctedFormat sub struct provides us the dimensions of the
     * corrected surface.
     */
    surface = SDL_CreateRGBSurface(0, screen->correctedFormat.width, screen->correctedFormat.height, 24, 0, 0, 0, 0);
    
    /* Set the corrected RGB pixels pointer of the conversion struct to the SDL pixel surface */
    amiVideo_setScreenCorrectedPixelsPointer(&screen, surface->pixels, surface->pitch, 4, TRUE, surface->format->Rshift, surface->format->Gshift, surface->format->Bshift, surface->format->Ashift);
    
    /* Convert the bitplanes to corrected RGB pixels */
    if(SDL_MUSTLOCK(surface) && SDL_LockSurface(surface) != 0)
    {
        fprintf(stderr, "Cannot lock the surface!\n");
        return 1;
    }
    
    amiVideo_convertScreenBitplanesToCorrectedRGBPixels(&screen);
    
    if(SDL_MUSTLOCK(surface))
        SDL_UnlockSurface(surface);

Cleaning up the screen conversion struct
----------------------------------------
After conversions have been performed, we may remove the converstion struct's
properties from memory:

    amiVideo_cleanupScreen(&screen);

Converting chunky graphics to planar graphics
=============================================
Apart from converting planar graphics to chunky or RGB format, we can also
perform the opposite process -- converting graphics in chunky format to bitplanes
so that a PC image can be displayed on a real Amiga with an OCS, ECS or AGA
chipset.

Acquiring a chunky graphics surface
-----------------------------------
In the examples used in the following subsections, we use the following
imaginary struct representing a chunky graphics surface with a palette:

    struct
    {
        UBYTE r, g, b, a;
    }
    Color;

    struct
    {
        struct Color *colors;
        unsigned int numOfColors;
        int width, height;
        void *pixels;
    }
    chunkyImage;

The above struct contains a palette with a specific amount of colors, a width
(in pixels), height (in scanlines) and an array containing bytes representing
chunky pixels.

Creating a conversion struct instance
-------------------------------------
To convert the chunky image surface to a planar graphics surface, we must create
an instance of the screen conversion struct taking the appropriate values of
the chunky image:

    /* Create a screen conversion struct */
    amiVideo_Screen conversionScreen;
    
    /*
     * Initialise a screen conversion struct. Chunky graphics normally have 8 bit
     * color components and 256 colors.
     */
    amiVideo_initScreen(&conversionScreen, chunkyImage.width, chunkyImage.height, 8, 8, 0);
    
    /* We must also set the colors of the viewport's chunky palette */
    amiVideo_setChunkyPaletteColors(&screen->palette, chunkyImage.colors, chunkyImage.numOfColors);
    
    /* Sets the uncorrected chunky pixels pointer of the conversion struct to that of the chunky pixel surface */
    amiVideo_setScreenUncorrectedChunkyPixelsPointer(conversionScreen, (amiVideo_UByte*)chunkyImage.pixels, chunkyImage.width);

After having configured the screen adapter, we can use it to convert the chunky
pixel surface to something that can be displayed on Amiga hardware. In the next
sections, we explain how this can be done.

Creating a screen and setting the bitplane pointers
---------------------------------------------------
In this example, we will use the AmigaOS graphics and intuition APIs to create a
custom intuition screen with the same dimensions and equivalent properties:

    #include <exec/types.h>
    
    #include <graphics/gfx.h>
    #include <intuition/intuition.h>
    
    #include <clib/graphics_protos.h>
    #include <clib/intuition_protos.h>
    
    /* Calculate a suitable viewport mode best suitable for displaying the image */
    ULONG viewportMode = amiVideo_calculateScreenViewportMode(&conversionScreen);
    
    /* Create an intuition screen */
    
    UWORD pens[] = { ~0 };
    
    struct TagItem screenTags[] = {
        {SA_Width, conversionScreen.width },
        {SA_Height, conversionScreen.height },
        {SA_Depth, 8, /* VGA chunky graphics have 256 colors */
        {SA_Title, "My converted picture"},
        {SA_Pens, pens},
        {SA_FullPalette, TRUE},
        {SA_Type, CUSTOMSCREEN},
        {SA_DisplayID, viewportMode},
        {SA_AutoScroll, TRUE},
        {TAG_DONE, NULL}
    };
    
    struct Screen *screen = OpenScreenTagList(NULL, screenTags);

    /* Set the bitplane pointers to those of the intuition screen */
    amiVideo_setScreenBitplanePointers(&conversionScreen, (amiVideo_UByte**)screen->ViewPort.RasInfo->BitMap->Planes);

Converting the palette
----------------------
The following example converts the colors of the chunky palette to the format
of the bitplanes:

    /* Convert the palette */
    amiVideo_convertChunkyColorsToBitplaneFormat(&conversionScreen.palette);

Setting the palette colors for OCS/ECS chipsets
-----------------------------------------------
There are two ways to set a palette on AmigaOS. Each option uses a different
color specification array.

The most trivial is the format used by the `LoadRGB4()` function. This format can
be generated as follows:

    /* Generate the color specification */
    amiVideo_UWord *colorSpecs = amiVideo_generateRGB4ColorSpecs(&conversionScreen);
    
    /* Set the palette using the generated color specification */
    LoadRGB4(screen->ViewPort, colorSpecs, conversionScreen.palette.bitplaneFormat.numOfColors);
    
    /* Remove the color specification from memory, since we don't need it anymore */
    free(colorSpecs);

The limitation of `LoadRGB4()` is that it can only set color values that have
4-bit color components, preventing someone to use the more advanced AGA chipset's
capabilities.

Setting the palette colors for AGA chipsets
-------------------------------------------
The other format is used by the `LoadRGB32()` function and can be generated as
follows:

    /* Generate the color specification */
    amiVideo_ULong *colorSpecs = amiVideo_generateRGB32ColorSpecs(&screen);
    
    /* Set the palette using the generated color specification */
    LoadRGB32(screen->ViewPort, colorSpecs);
    
    /* Remove the color specification from memory, since we don't need it anymore */
    free(colorSpecs);

The `LoadRGB32()` function can also be used to set colors with 8-bit color
components.

Converting chunky pixels to bitplanes
-------------------------------------
To convert the chunky pixels to bitplanes, we can do:

    /* Convert chunky pixels to bitplanes */
    amiVideo_convertScreenChunkyPixelsToBitplanes(&conversionScreen);

Cleaning up the screen conversion struct
----------------------------------------
After performing a conversion, we may remove the converstion struct's properties
from memory once we don't need them anymore:

    amiVideo_cleanupScreen(&conversionScreen);

Miscellaneous functions
=======================
In the previous code examples, we have used various fixed values for certain
properties. This library also provides a number of functions that can auto select
them for you.

Auto selecting a suitable color format
--------------------------------------
    amiVideo_ColorFormat format;
    amiVideo_Screen screen;
    screen.viewportMode = AMIVIDEO_VIDEOPORTMODE_HAM;
    
    /* Returns AMIVIDEO_RGB_FORMAT */
    format = amiVideo_autoSelectColorFormat(&screen);

The above function invocation auto selects the most memory efficient color format
for a given viewport mode. In the example, it returns AMIVIDEO_RGB_FORMAT since
we have to display more than 256 colors.

Auto selecting a lowres pixel scale factor
------------------------------------------
    /* Returns 4 */
    amiVideo_autoSelectLowresPixelScaleFactor(AMIVIDEO_VIDEOPORTMODE_SUPERHIRES);

This function invocation auto selects the most memory efficient lowres pixel
scale factor for a given viewport mode. The above example needs 4 bytes for a
lowres pixel since super hires screens are at least 1280 pixels per scanline.

Auto selecting a viewport mode
------------------------------
    /* Returns AMIVIDEO_VIDEOPORTMODE_SUPERHIRES | AMIVIDEO_VIDEOPORTMODE_LACE */
    amiVideo_autoSelectViewportMode(1280, 512);

The above function invocation auto selects the best suitable resolution viewport
mode bits for a screen with the given dimensions. To properly display an 1280x512
image on an Amiga display, we have to use a interlaced screen with super hires
resolution.

Installation on Unix-like systems
=================================
Compilation and installation of this library on Unix-like systems is straight
forward, by using the standard GNU autotools build instructions:

    $ ./configure
    $ make
    $ make install

More details about the installation process can be found in the `INSTALL` file
included in this package.

Building with Visual C++
========================
This package can also be built with Visual C++ for Windows platforms. The
solution file resides in `src/libamivideo.sln` that can be used in Visual Studio
to edit and build it.

Alternatively, you can also build it with `MSBuild` from the command-line:

    $ MSBuild libamivideo.sln

The output is produced in the `Debug/` directory.

License
=======
This library is available under the MIT license
