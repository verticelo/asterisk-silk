# Asterisk patch for SILK

To add a codec for SIP/SDP (m=, rtmap, and ftmp), you create a format module in Asterisk: `codec_silk.patch` (for m= and rtmap) and `res/res_format_attr_silk.c` (for fmtp). However, this requires both call legs to support SILK (pass-through only). If one leg does not support SILK, the call has no audio. Or, if you use the pre-recorded voice and music files of Asterisk, these files cannot be heard, because they are not in SILK but in signed linear (SLN). Therefore, this repository adds not just a format module but a transcoding module as well: `codecs/codec_silk.c`.

The ZIP file for SILK contains the latest RTP payload-format description in the folder `doc`. This repository here is an wrapper implementation of that document. SILK is [deprecated](http://blogs.skype.com/2012/09/12/skype-and-a-new-audio-codec/) since September 2012 because its work continues within the [Opus Codec](http://tools.ietf.org/html/rfc7587) in the mode Linear Prediction ([LP](http://wiki.xiph.org/OpusFAQ#Is_the_SILK_part_of_Opus_compatible_with_the_SILK_implementation_shipped_in_Skype.3F)). Nevertheless, you are here because you have a VoIP client which does not support Opus yet but still supports SILK. Or you want to learn more about SILK because it is one foundation of Opus. Research papers comparing SILK with other audio codecs: InterSpeech [2010](http://research.nokia.com/files/public/%5B12%5D_Interspeech%202010_Voice%20Quality%20Evaluation%20of%20Recent%20Open%20Source%20Codecs.pdf) and [2011](http://research.nokia.com/files/public/%5B16%5D_InterSpeech2011_Voice_Quality_Characterization_of_IETF_Opus_Codec.pdf). Further [examples…](http://www.opus-codec.org/examples/)

This patch is for Asterisk 13. If you use Asterisk 11, please, consider the [binaries created by Digium](http://www.digium.com/products/asterisk/downloads) or [this…](http://github.com/mordak/codec_silk).

## Installing the patch

The patch was built on top of Asterisk 13.6.0. If you use a newer version and the patch fails, please, [report](http://help.github.com/articles/creating-an-issue/)!

    cd /usr/src/
    wget downloads.asterisk.org/pub/telephony/asterisk/asterisk-13-current.tar.gz
    tar zxf ./asterisk*
    cd ./asterisk*
    sudo apt-get --assume-yes install build-essential autoconf libssl-dev libncurses-dev libnewt-dev libxml2-dev libsqlite3-dev uuid-dev libjansson-dev libblocksruntime-dev

Install libraries:

If you do not want transcoding but pass-through only (because of license issues) please, skip this step. To support transcoding, you’ll need to compile SILK, for example the floating point variant (FLP) in Debian/Ubuntu:

    cd /usr/src/
    wget web.archive.org/web/20130205164531/http://developer.skype.com/silk/SILK_SDK_SRC_v1.0.9.zip
    unzip -qq SILK*
    cd ./SILK_SDK_SRC_FLP*
    CFLAGS='-fPIC' make lib
    sudo cp ./interface/*.h /usr/include/
    sudo cp ./libSKP_SILK_SDK.a /usr/lib/

Apply all patches:

    wget github.com/traud/asterisk-silk/archive/master.zip
    unzip -qq master.zip
    rm master.zip
    cp --verbose --recursive ./asterisk-silk*/* ./
    patch -p0 <./codec_silk.patch
    patch -p0 <./build_silk.patch

Run the bootstrap script to re-generate configure:

    ./bootstrap.sh

Configure your patched Asterisk:

    ./configure

Enable SLN16 in menuselect for transcoding, for example via:

    make menuselect.makeopts
    ./menuselect/menuselect --enable-category MENUSELECT_CORE_SOUNDS

Compile and install:

    make --jobs
    sudo make install

## Testing
You can test SILK out of the box using:

A.  (Google Android) [CSipSimple](http://play.google.com/store/apps/details?id=com.csipsimple)

B.  (Google Android) [CounterPath Bria](http://play.google.com/store/apps/details?id=com.bria.voip)

C.  (Apple iOS) [CounterPath Bria](http://itunes.apple.com/app/bria-iphone-edition-voip-softphone/id373968636)

D.  (Windows Phone 8) [Linphone](http://www.windowsphone.com/s?appId=99661466-8c5c-489b-a567-569c1f480d29)

However, these VoIP/SIP clients offer G.722 and Opus, which should be used for wide-band audio instead. G.722 transcoding is build into Asterisk already. Opus can be added as [transcoding module…](http://github.com/seanbright/asterisk-opus/)

## What is missing
`codecs.conf` is not used, because the client should tell the server (Asterisk) which configuration is desired. However, tests revealed that none of the above SILK implementations send the format attribute `maxaveragebitrate`, although each implementation is using a rather low bit-rate actually. As consequence, the client is sending with a rather moderate quality and Asterisk is responding with the highest quality possible. This wastes bandwidth. If you are not able to convince the author of that VoIP client to send its `maxaveragebitrate` via SIP/SDP, you have to edit the source of `res/res_format_attr_silk.c` and `codecs/codec_silk.c`. If you need not a general but a configuration for each peer, please, [add](http://help.github.com/articles/using-pull-requests/) or [report](http://help.github.com/articles/creating-an-issue/) support for `codecs.conf`.

Currently, the bit-rate is fixed and does not change while in a call, because there is no connection between RTCP and SILK. Therefore, it is not possible to adjust the bit-rate for congestion control. For example, when a mobile network is used, the user might travel from very fast 4G (LTE) to a fast 3.5G (HSPA), dropping down to a moderate 3G (UMTS) or even near to unusable 2G (GSM) connection. Same applies for the [Opus module](http://lists.digium.com/pipermail/asterisk-dev/2015-January/072534.html), [FreeSWITCH](http://lists.freeswitch.org/pipermail/freeswitch-users/2014-July/106346.html), and [PJSIP](http://lists.pjsip.org/pipermail/pjsip_lists.pjsip.org/2013-February/015772.html).

Several frames per payload (compound payload) are another issue, for example when the packetization time (`ptime`) is longer than 20 ms. For sending, `codecs/codec_silk.c:lintosilk_frameout` would need an change and `main/codec_builtin.c:silk_samples` while receiving.

## Thanks goes to
* Skype for creating SILK and its library.
* Asterisk team: Thanks to their efforts and architecture this module was adopted in one working day.
* [Todd Mortimer](http://lists.digium.com/pipermail/asterisk-bsd/2012-April/003920.html) provided the starting point with his module for Asterisk 11.