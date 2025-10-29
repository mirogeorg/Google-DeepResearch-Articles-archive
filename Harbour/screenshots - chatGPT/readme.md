You: 

i need to convert screenshots from PNG to video file for archiving purposes losslessly. I need most efficient encoder for screenshots with little deltas. Space efficiency is the most important factor. research even custom rarely used codecs.

ChatGPT: 

To make sure I find the most space-efficient and lossless video encoder for your screenshots, could you please clarify a few things:

1.  What is the resolution and bit depth (8-bit, 10-bit, etc.) of your PNG screenshots?
    
2.  How many screenshots are you typically encoding per video (ballpark estimate)?
    
3.  What is your preferred output container format (e.g., .mkv, .mp4, .avi, or no preference)?
    
4.  Are you open to using command-line tools or custom software that requires compilation, or do you need something plug-and-play?
    
5.  Is decoding performance or compatibility with common players (e.g., VLC) a concern, or is it purely about storage efficiency?
    

Once I have this info, I’ll dig deep into both mainstream and niche encoders that are optimized for lossless compression of screen content.

You: 

1\. FullHD 8-bit depth 2. few hundreds 3. no preference 4. command line is requirement 5. compatibility and performance is of no concern

ChatGPT: 

Thanks! I’ll research the most space-efficient, lossless video encoders tailored for archiving PNG screenshots at Full HD resolution (8-bit), where frame differences are minimal and compression efficiency is the top priority. I’ll focus on both mainstream and custom/niche codecs that work via command-line tools and don't prioritize compatibility or decoding performance.

I’ll let you know once I’ve found the best options for your use case.

Research completed in 15m · 25 sources · 121 searches

Best Lossless Video Encoder for PNG Screenshot Archiving

# Space-Efficient Lossless Video Encoders for Full-HD Screenshots

## Introduction

Archiving a sequence of Full HD (1920×1080) screenshots with minimal frame-to-frame changes calls for **lossless video encoders** that maximize compression. In this scenario, typical concerns like playback compatibility or encoding/decoding speed are secondary to achieving the smallest file size. The key to high compression is exploiting the tiny “deltas” between frames using inter-frame compression (where possible) and using codecs tailored for screen content (with large flat-color areas and sharp text). Below we explore both mainstream and niche lossless video codecs, highlighting those proven to yield superior compression on near-static image sequences. We also provide example command-line usages and discuss pros/cons for each option.

## Mainstream Lossless Video Codecs

These are widely-used video codecs that offer a lossless mode. They benefit from advanced compression techniques (motion prediction, entropy coding) and are readily usable via tools like FFmpeg.

### H.264 (x264) – Lossless Mode

H.264 can be used in a true lossless configuration (no quantization), typically via the x264 encoder. Despite its age, x264 remains extremely competitive for compressing clean, static content. In fact, tests have found x264’s lossless mode often matches or even outperforms newer codecs for “clean” (low-noise, simple) sources[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=AV1%20sucks%20for%20lossless)[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=,used%20with%20the%20fastdecode%20tune). It uses inter-frames to great effect: if parts of a frame repeat from the previous frame, x264 will encode them as “skipped” blocks costing only a few bytes. To preserve full 8-bit RGB detail, x264 offers a High 4:4:4 Predictive profile (no chroma subsampling) and even an RGB mode.

-   **Pros:** Mature and widely supported. Achieves excellent compression on static or near-static footage – x264 was *“the king of compression on clean clips”* in one comparison[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=,used%20with%20the%20fastdecode%20tune). Decoding is relatively fast and lightweight.
    
-   **Cons:** Lacks explicit screen-content optimizations (no special handling of text or GUI visuals beyond normal intra prediction). Newer codecs like AV1 or HEVC can theoretically save a few extra percent, but only with vastly slower encoding[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=aomenc,4%20or%20slower).
    
-   **Usage Example:** Using FFmpeg with x264’s lossless mode:
    
    ```bash
    ffmpeg -framerate 30 -i frame_%04d.png -c:v libx264 -preset veryslow -qp 0 -pix_fmt yuv444p output.mkv
    ```
    
    In this command, `-qp 0` (or equivalently `-crf 0`) and the use of yuv444p ensure lossless compression (High 4:4:4 profile). For **pixel-perfect RGB** without any color conversion, use the `libx264rgb` encoder variant instead of the default (which avoids RGB→YUV conversion entirely)[superuser.com](https://superuser.com/questions/1429256/producing-lossless-video-from-set-of-png-images-using-ffmpeg#:~:text=You%27re%20using%20a%20generic%20value,or%20libx265%29%20instead). For maximum compression on static images, you can also use a large GOP size (e.g. `-g 300` or more) to allow long stretches of P-frames referencing the first frame.
    

### H.265 (HEVC, x265) – Lossless Mode

HEVC offers a lossless mode as well, accessible via x265. HEVC’s more advanced intra and inter coding tools can reduce file size further in some cases. Notably, the HEVC standard has an extension for Screen Content Coding (SCC) – including features like Intra Block Copy (copying blocks *within* the same frame) and explicit text/graphic modes – which significantly improve compression of computer-generated imagery[multicorewareinc.com](https://multicorewareinc.com/enhancing-x265-with-hevc-screen-content-coding/#:~:text=Screen%20Content%20Coding%20,to%20improve%20efficiency%20when%20encoding)[multicorewareinc.com](https://multicorewareinc.com/enhancing-x265-with-hevc-screen-content-coding/#:~:text=SCC%20introduces%20specialized%20tools%20like,time%20collaboration%2C%20video%20conferencing%2C%20remote). As of x265 v4.x, support for SCC features is available when enabled at build time[en.wikipedia.org](https://en.wikipedia.org/wiki/List_of_codecs#:~:text=,517%20Lawrence%20Berkeley%20National), making it very effective for compressing slides, GUI screenshots, and other synthetic images. Even without SCC, x265 can be run in lossless mode to leverage its improved entropy coding and prediction. However, real-world tests suggest that for typical screenshot content, x265’s gains over x264 are modest – and in some cases x264 can produce smaller files[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=AV1%20sucks%20for%20lossless) (this can happen due to encoder heuristics and overhead). Use x265 if you are seeking every last bit of compression and are willing to accept slower encoding.

-   **Pros:** More modern codec with the potential for better compression than H.264. Handles flat areas and repetitive patterns well, especially if **Screen Content Coding** tools are enabled (dramatically boosts efficiency for text and graphics)[multicorewareinc.com](https://multicorewareinc.com/enhancing-x265-with-hevc-screen-content-coding/#:~:text=Screen%20Content%20Coding%20,to%20improve%20efficiency%20when%20encoding)[multicorewareinc.com](https://multicorewareinc.com/enhancing-x265-with-hevc-screen-content-coding/#:~:text=SCC%20introduces%20specialized%20tools%20like,time%20collaboration%2C%20video%20conferencing%2C%20remote). Still uses inter-frame compression to capitalize on minimal frame differences.
    
-   **Cons:** Encoding is slower and memory-intensive. Decoding is also more CPU-demanding (though not a primary concern for archiving). Without SCC mode, the size improvement over x264 is small – one Reddit test found x265 lossless was actually slightly larger than x264 for certain clean content[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=AV1%20sucks%20for%20lossless). Player support is not as ubiquitous as H.264 (if that matters at all).
    
-   **Usage Example:** Using FFmpeg with x265 in lossless mode:
    
    ```bash
    ffmpeg -i frame_%04d.png -c:v libx265 -preset slow -x265-params lossless=1 output.mkv
    ```
    
    Here the `lossless=1` parameter engages true lossless encoding (no quantization)[stackoverflow.com](https://stackoverflow.com/questions/37344997/how-to-get-a-lossless-encoding-with-ffmpeg-libx265#:~:text=Why%20are%20all%20of%20those,265). You do not need to specify `-crf` or `-qp` when using `lossless=1`. Ensure you use a 4:4:4 pixel format if preserving exact color values (e.g., add `-pix_fmt yuv444p` for full-resolution color, since the default HEVC profile would be 4:2:0 which is *not* lossless for RGB inputs). If you have a custom build of x265 with Screen Content Coding enabled, you can add `--scc 1` in `-x265-params` to turn on those extra compression tools for screen content, which can further reduce size on graphical material.
    

### AV1 – Lossless Mode (libaom-AV1)

AV1 is a next-generation codec that, while typically used for lossy web video, also supports a lossless mode. In lossless configuration, AV1 can compress slightly better than H.264/H.265, especially when using its most exhaustive encoding settings[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=aomenc,4%20or%20slower). It incorporates some screen-friendly features by default (such as Intra Block Copy and a palette mode similar to GIF/WebP) and has very efficient entropy coding. That said, the improvement in file size is relatively small – typically a few percent smaller than x264/x265 in tests – and comes at **massive cost in encoding time**[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=aomenc,4%20or%20slower). AV1 encoding on the slowest presets is extremely CPU-intensive, which might be impractical for “a few hundred frames” unless you have significant computing power or time. Also, not all AV1 encoders currently support lossless mode (the reference *libaom* does, while some faster alternatives like SVT-AV1 only added lossless support recently)[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=BlueSwordM).

-   **Pros:** **Highest compression** potential among mainstream codecs for lossless video[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=aomenc,4%20or%20slower). Designed with modern techniques that excel on both natural and synthetic content. Supports features like tile-based decoding and can handle alpha channel via AVIF (image sequence) or extended AV1 features. An AV1 lossless encode will squeeze a bit more out of static screenshots than H.265 or H.264 can.
    
-   **Cons:** **Encoding speed is the major drawback.** Achieving those extra bytes savings requires using slow encoding modes – one user noted AV1 only pulled ahead of x264/x265 after using very slow settings (CPU-used=4 or lower)[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=aomenc,4%20or%20slower). Decoding is also slower and not as widely supported in consumer tools yet. The returns diminish rapidly: you might get, say, 5% smaller files for orders of magnitude more encoding time, so it may *“not be worth it for the amount of computations spent for a small compression gain.”*[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=aomenc,4%20or%20slower)
    
-   **Usage Example:** Using FFmpeg with libaom-AV1 in lossless mode:
    
    ```bash
    ffmpeg -i frame_%04d.png -c:v libaom-av1 -crf 0 -b:v 0 -aom-params lossless=1:aq-mode=0 output.mkv
    ```
    
    Here `lossless=1` forces true lossless encoding (perfect reconstruction of input)[aomedia.googlesource.com](https://aomedia.googlesource.com/av1-xbox-one/+/refs/heads/main/libav1/aom/aomcx.h#:~:text=%2F,lossless%20coding%20mode). We set `-crf 0 -b:v 0` to effectively give the encoder free rein on bitrate. Note that you should use the **lowest speed preset** you can tolerate (libaom’s default is very slow already, but `-cpu-used` can adjust speed). For example, add `-cpu-used 4` (or 3, 2, etc.) to trade speed for compression – lower values = better compression. Keep in mind that even at the slowest settings, AV1’s file size improvements might only be a few percent versus x264[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=Just%20tried%2C%20I%20got%20around,improvement%20with%20x264%20over%20x265), so judge based on your priorities.
    

### VP9 – Lossless Mode (libvpx)

Google’s VP9 (the video codec behind WebM) also supports a lossless mode and is worth considering for screen capture compression. In lossless VP9, the codec activates an alternate coding path akin to WebP lossless images (which can involve techniques like color indexing and LZ77 compression for large flat areas)[ffmpeg.org](https://ffmpeg.org/ffmpeg-codecs.html#:~:text=lossy%20or%20lossless%20mode,separate%20codec%20developed%20by%20Google)[ffmpeg.org](https://ffmpeg.org/ffmpeg-codecs.html#:~:text=screen). VP9’s efficiency is generally between H.264 and HEVC; it was designed more for lossy streaming, but its lossless capability coupled with certain encoding tunes can work well for graphics. VP9 has a **screen content mode** (`-tune-content screen`) which, when combined with lossless, optimizes the compression for GUI/text scenes (for example, it enables special intra prediction and possibly palette modes for large single-color regions). VP9 can also handle an alpha channel in a WebM container (via the VP9 Transparency extension), which might be useful if your PNGs have transparency.

-   **Pros:** Free and open-source, with broad browser support (if that ever matters). Offers reasonably good compression on screenshots – it can achieve smaller files than x264 in many cases, though not quite matching AV1/HEVC levels. Has dedicated screen-content tuning that improves compression for slides, animations, or desktop recordings[ffmpeg.org](https://ffmpeg.org/ffmpeg-codecs.html#:~:text=screen). Supports 8-bit RGB (internally converted to 4:4:4 YUV or YUVA for alpha) without quality loss when using lossless mode.
    
-   **Cons:** Encoding speed is slower than x264 (but usually faster than AV1). The compression gain over x264 lossless isn’t huge for photographic content; for very simple graphics, VP9 and x264 might each sometimes win on different files. Decoding VP9 is computationally heavier than H.264 and not as universally accelerated.
    
-   **Usage Example:** Using FFmpeg with libvpx-vp9 in lossless screen-content mode:
    
    ```bash
    ffmpeg -i frame_%04d.png -c:v libvpx-vp9 -lossless 1 -tune content=screen -pix_fmt yuv444p output.webm
    ```
    
    This will produce a lossless VP9 WebM. The `-lossless 1` flag turns on VP9’s lossless coding path[ffmpeg.org](https://ffmpeg.org/ffmpeg-codecs.html#:~:text=Screen%20content%20mode%2C%20one%20of%3A,with%20more%20aggressive%20rate%20control). We also specify `-tune content=screen` (FFmpeg’s way to tune for screen content) to leverage any screen-specific optimizations. Ensure `yuv444p` is used to avoid chroma loss. If an alpha channel is present, use `-pix_fmt yuva444p` and add `-auto-alt-ref 0` (and potentially `-lag-in-frames 0`) since VP9 alt-ref frames don’t support alpha well. Note that lossless VP9 doesn’t need a `-crf` value – it will use full quality; you might however set `-cpu-used` to a higher number (0 is slowest, 5 or 6 for faster encodes) depending on your speed/size trade-off preference.
    

## Specialized or Niche Lossless Codecs

Beyond the common codecs above, there are several specialized lossless video codecs, some of which were designed with screen recording or maximum compression in mind. These may not achieve the absolute smallest files compared to inter-frame codecs like x264, but they can shine in certain use cases. Many of them are **intra-frame (keyframe-only) codecs**, meaning they compress each frame independently – this is less efficient for sequences of very similar frames, but can be desirable for simplicity or specific archival needs. Below are a few notable ones:

### FFV1 (FFmpeg Video Codec #1)

**FFV1** is an open, lossless intra-frame codec developed as part of FFmpeg, often used in digital video preservation. It employs *context-adaptive arithmetic coding* and efficient prediction, making it extremely good at compressing images without loss[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=Lagarith%20is%20a%20lossless%20video,Pixel%20values%20are%20first)[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=very%20close%20to%20the%20entropy,use%20the%20previous%20frame%20again). FFV1 is known to produce compression ratios comparable to or better than JPEG2000 (another archival format) but at a fraction of the computational cost[en.wikipedia.org](https://en.wikipedia.org/wiki/FFV1#:~:text=FFV1%20,4)[en.wikipedia.org](https://en.wikipedia.org/wiki/FFV1#:~:text=FFV1%20proved%20to%20be%20a,14). Because it’s intra-only, it won’t exploit frame-to-frame redundancy; however, for “mostly static” content, each frame is so similar that FFV1’s entropy coder will still compress repeated patterns in each frame very well. In practice, FFV1 often achieves smaller sizes than older lossless codecs like HuffYUV or even Lagarith for the same content[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=Lagarith%20is%20a%20lossless%20video,Pixel%20values%20are%20first). It has become a **preferred archival codec** for institutions like the U.S. Library of Congress and many libraries/archives[en.wikipedia.org](https://en.wikipedia.org/wiki/FFV1#:~:text=)[en.wikipedia.org](https://en.wikipedia.org/wiki/FFV1#:~:text=FFV1%20proved%20to%20be%20a,14) due to its openness, compression, and error resilience.

-   **Pros:** **High compression efficiency** (especially for intra-only): FFV1 was shown to compress YUV video ~3–5% better than Lagarith in one comparison[forum.videohelp.com](https://forum.videohelp.com/threads/318382-Lagarith-or-x264-lossless#:~:text=,since%20ffdshow%20and), and it consistently outperforms older codecs like HuffYUV. Handles grain, noise, and flat graphics well via adaptive entropy coding. Open standard (now standardized as RFC 9043) with broad tool support[en.wikipedia.org](https://en.wikipedia.org/wiki/FFV1#:~:text=Video%20archiving)[en.wikipedia.org](https://en.wikipedia.org/wiki/FFV1#:~:text=FFV1%20proved%20to%20be%20a,14). Suitable for long-term archiving (robustness and parity data options, via Matroska container).
    
-   **Cons:** Intra-frame only: it doesn’t utilize inter-frame compression, so it may not shrink *quite* as much as inter-frame codecs on sequences with lots of unchanged areas. Encoding/decoding is CPU-intensive (slower to decode than, say, x264), which might be a factor if you need to frequently access the archived video. Not as widely known outside archival circles, so some consumer media players might not play FFV1 by default (but it’s easily decodable with FFmpeg/LAV Filters).
    
-   **Usage:** FFV1 is available through FFmpeg. For example:
    
    ```bash
    ffmpeg -i frame_%04d.png -c:v ffv1 -level 3 -g 1 output.mkv
    ```
    
    This uses FFV1 (default level 3 = latest version) in an MKV container. `-g 1` forces every frame to be a keyframe (though FFV1 doesn’t support inter coding anyway, so it’s intra-only by design). The output will be mathematically identical to the input frames, and typically much smaller than a series of raw or PNG images. Expect FFV1 to compress slightly better (with slower decoding) than Lagarith or other old VFW codecs[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=Lagarith%20is%20a%20lossless%20video,Pixel%20values%20are%20first) – it “delivers the highest possible image resolution and color quality” while still reducing file size[blogs.library.duke.edu](https://blogs.library.duke.edu/bitstreams/2021/03/12/ffv1-the-gains-of-lossless/#:~:text=FFV1%3A%20The%20Gains%20of%20Lossless,resolution%2C%20color%20quality%20and).
    

### Lagarith Lossless Codec

**Lagarith** is a classic lossless video codec (Windows VFW) that was *“intended for editing and archiving”*. It improves upon older codecs like HuffYUV by using smarter compression: median prediction of pixels, optional run-length encoding, and **arithmetic coding** instead of Huffman coding[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=the%20fact%20that%20it%20uses,why%20Lagarith%20can%20compress%20simple). Importantly for static content, Lagarith supports **“null frames”**[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=very%20close%20to%20the%20entropy,use%20the%20previous%20frame%20again) – if a frame is identical to the previous one, it doesn’t store the frame again, just a marker indicating a duplicate frame. This can drastically save space when there are sections of no change. Even when frames aren’t identical, Lagarith’s entropy coding is very effective at compressing large flat areas or repeated patterns in screenshots. In practice, Lagarith often yields smaller files than HuffYUV, Alparysoft, CorePNG and other contemporaries[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=Lagarith%20is%20a%20lossless%20video,RGB32%2C%20RGBA%2C%20YUY2%2C%20and%20YV12). Only a few codecs compress better (notably FFV1 and the proprietary MSU codec), though those tend to be slower[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=Lagarith%20is%20a%20lossless%20video,Pixel%20values%20are%20first). Lagarith was popular in the 2000s for video capture and archival on Windows.

-   **Pros:** Very good compression on screen content – significantly outperforms HuffYUV on highly static scenes[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=able%20to%20operate%20in%20several,modified%20Run%20Length%20Encoding%20if). The combination of median predictor + run-length + arithmetic coding means it gets *“very close to the entropy of the data”*, so it **“can compress simple frames much better than Huffyuv, and avoid expanding high-static video.”**[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=Lagarith%2C%20the%20byte%20stream%20may,use%20the%20previous%20frame%20again) Duplicate-frame suppression (null frames) is a big plus for sequences with occasional identical frames (saving 100% of those frames’ size)[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=very%20close%20to%20the%20entropy,use%20the%20previous%20frame%20again). Supports multiple color formats (RGB24/32, RGBA, YUY2, YV12), giving flexibility for different sources[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=able%20to%20operate%20in%20several,In).
    
-   **Cons:** Being a VFW codec, it’s mostly Windows-only and not as straightforward to use on the command line. It’s also not actively developed (last update was in 2011). Encoding/decoding speed is modest – faster than FFV1, but slower than real-time in high resolutions. Limited multithreading (though it added some in later versions). For purely static image sequences, modern interframe codecs (x264, etc.) will still beat Lagarith in size, since Lagarith doesn’t do inter prediction (except for identical-frame skipping).
    
-   **Usage:** Lagarith can be used in VirtualDub or via FFmpeg (FFmpeg can decode Lagarith streams and, on Windows, can use it via AVI output or VfW interface). A typical use is to install Lagarith and then in VirtualDub select Lagarith for compression. In FFmpeg, you can request Lagarith with `-c:v lagarith` (if the codec is properly installed in your Windows system). For example:
    
    ```batch
    ffmpeg -i frames_%03d.png -c:v lagarith output.avi
    ```
    
    This would produce a Lagarith-compressed AVI. Be sure to enable “Prevent upsampling” and set the mode to match your source color space in Lagarith’s settings if using its configuration UI, to avoid any color shifts. Given its age and platform dependency, Lagarith is recommended if you already use it in a Windows workflow, but otherwise newer open solutions (like FFV1 or MagicYUV) might be preferable.
    

### Other Noteworthy Codecs:

-   **HuffYUV and FFVHuff:** HuffYUV is a very fast, older lossless codec using Huffman coding. FFVHuff is an FFmpeg implementation with minor improvements. They encode each frame with zero loss but achieve only mild compression (often 2:1 to 5:1) depending on content. For screenshots with large solid areas, HuffYUV might not compress as well as newer methods – Lagarith’s author notes *“for high static scenes… Lagarith significantly outperforms Huffyuv”*[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=able%20to%20operate%20in%20several,modified%20Run%20Length%20Encoding%20if). Use HuffYUV if you need lightning-fast encoding/decoding and are willing to accept larger files. Command-line usage: `-c:v huffyuv` (or `ffvhuff`). These support YUV 4:2:2 and RGB; ensure you pick a mode with no chroma subsampling for true lossless archive of screenshots.
    
-   **Ut Video:** An open source lossless codec, often used in video editing. It’s intra-frame and performs LZ-based compression on prediction residuals. Ut Video offers multithreaded encoding and decoding, making it faster on multi-core systems. Compression is similar to HuffYUV/FFVHuff or a bit better, but typically not as tight as Lagarith or x264. It supports YUV 4:2:0, 4:2:2, 4:4:4, and RGBA (Ut Video RGBA mode is useful for lossless with alpha). Use it via FFmpeg with `-c:v utvideo`. Pros: cross-platform, very fast. Cons: larger output sizes (not optimized for maximum compression of static content).
    
-   **MagicYUV:** A newer entrant focused on being a fast, efficient lossless codec. MagicYUV is designed for high resolutions and offers excellent speed, with compression better than HuffYUV and comparable or slightly better than Ut Video. It supports 4:4:4 and alpha as well. While MagicYUV is proprietary, it’s free to use. If speed is a factor during the archival process (e.g., capturing hundreds of frames in real-time), MagicYUV is a good choice; however, in terms of ultimate file size, x264 or FFV1 might still win by a margin. MagicYUV can be invoked in FFmpeg (`-c:v magicyuv`) if you have the codec library set up.
    
-   **PNG Frames in MKV/AVI:** It’s worth noting you can simply store each PNG image in a video stream (each frame encoded as a PNG, which is **intra-frame** compression only). This is essentially like zipping the images together in one file. FFmpeg supports `-c:v png` to create such a video. The benefit is that PNG is great for compressing graphics with few colors; however, because there is no inter-frame compression, the total size will be roughly the sum of all PNG frame sizes (and you gain little from the video container, aside from easier playback in some players). In fact, doing `-c:v copy` from a sequence of PNGs into MKV is possible (copies the PNG data as frames)[superuser.com](https://superuser.com/questions/1429256/producing-lossless-video-from-set-of-png-images-using-ffmpeg#:~:text=If%20you%20want%20to%20avoid,YUV%20colorspace%20conversion%20use%20libx264rgb), but many players won’t know how to handle a video stream of raw PNG frames[superuser.com](https://superuser.com/questions/1429256/producing-lossless-video-from-set-of-png-images-using-ffmpeg#:~:text=1). So, this approach is not recommended if minimizing size is the goal – you’d be better off compressing the sequence with 7-Zip in that case, or using a real video codec to exploit frame redundancy.
    
-   **Zip Motion Block Video (ZMBV):** ZMBV is a lossless codec specifically designed for screen recording (used by DOSBox). It works by dividing frames into blocks, performing an XOR (difference) against the previous frame for each block, and then compressing those differences with Deflate (zlib)[shouldiremoveit.com](https://www.shouldiremoveit.com/Zip-Motion-Block-Video-codec-Remove-Only-15297-program.aspx#:~:text=Zip%20Motion%20Block%20Video%20codec,encoding%20of%20low%20color%20content). This is very effective for *pixel-identical* regions between frames – unchanged areas compress to near-zero size – and fairly effective for small changes (the XOR’d output will be sparse, which zlib handles well). ZMBV is limited in that it doesn’t do motion compensation (so if something moves, it treats it as a difference rather than shifting a block), and it was originally meant for 8-bit palettized video, though it also supports 24-bit color. The upside is that it’s fast and simple. In scenarios with *very* little change per frame, ZMBV can produce surprisingly small files (since most blocks XOR to zero and compress tightly). FFmpeg can encode ZMBV with `-c:v zmbv`. If your screenshots are mostly identical and just occasionally change a small region (say a mouse cursor or a blinking icon), ZMBV might give compression on par with more complex codecs. Keep in mind it’s primarily optimized for older “low-color content” and may not be as efficient for full 1080p true-color images if lots of detail changes[shouldiremoveit.com](https://www.shouldiremoveit.com/Zip-Motion-Block-Video-codec-Remove-Only-15297-program.aspx#:~:text=The%20video%20portion%20of%20the,encoding%20of%20low%20color%20content). But for completeness, it is a viable archival codec for screen captures (especially retro-style or GUI content).
    

## Conclusion and Recommendations

For archiving Full HD screenshots with minimal differences, the **top choices** in practice are the modern inter-frame codecs in lossless mode – notably **x264** for a great balance of size and speed, or **x265** if you enable screen-content tools. x264’s zero-loss mode has proven very effective on static image sequences[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=,used%20with%20the%20fastdecode%20tune) and is straightforward to use via FFmpeg[superuser.com](https://superuser.com/questions/1429256/producing-lossless-video-from-set-of-png-images-using-ffmpeg#:~:text=You%27re%20using%20a%20generic%20value,or%20libx265%29%20instead). If you are willing to trade a lot of time for a bit more compression, **AV1 (libaom)** in lossless mode will squeeze out the smallest possible file at the cost of slow encoding[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=aomenc,4%20or%20slower).

Among intra-frame codecs, **FFV1** stands out for maximum compression – it’s well-suited to archival storage where decoding speed is less important, and it outperforms most older codecs in compression ratio[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=Lagarith%20is%20a%20lossless%20video,Pixel%20values%20are%20first). **Lagarith** is another strong option if working in a Windows environment, offering good compression (especially for large swathes of static content due to null frames)[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=very%20close%20to%20the%20entropy,use%20the%20previous%20frame%20again), though with less cross-platform ease. Fast codecs like HuffYUV, MagicYUV, or Ut Video are more about speed than size, so they’re less ideal if storage size is the only priority – still, they guarantee lossless quality and may be worth mentioning if you need quick encoding.

In summary, for **minimal storage size**: try x264 (lossless) or FFV1 first; if you need smaller and don’t mind slow, use x265 (lossless, with SCC if possible) or AV1. For completeness, a few niche screen-recording codecs (ZMBV, etc.) exist, but their compression gains on full-color HD screenshots will usually be eclipsed by x264/x265’s advanced algorithms. All the above options can be used via command-line tools like FFmpeg, ensuring a scriptable and reproducible archiving workflow. Be sure to verify the lossless output by decoding a few frames and comparing to the original PNGs pixel-for-pixel. With the right choice, you’ll achieve significant size reduction while preserving every bit of your screenshots.

**Sources:** The recommendations and examples above are based on codec documentation and user evaluations: for instance, x264’s superiority on clean static content[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=,used%20with%20the%20fastdecode%20tune), x265’s screen content coding enhancements[multicorewareinc.com](https://multicorewareinc.com/enhancing-x265-with-hevc-screen-content-coding/#:~:text=Screen%20Content%20Coding%20,to%20improve%20efficiency%20when%20encoding), and AV1’s marginal gains at high computational cost[reddit.com](https://www.reddit.com/r/AV1/comments/1g0gwvk/has_anyone_tested_av1_lossless_encoding_similar/#:~:text=aomenc,4%20or%20slower). Lagarith’s official documentation highlights its compression method and null-frame capability[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=the%20fact%20that%20it%20uses,why%20Lagarith%20can%20compress%20simple)[lags.leetcode.net](https://lags.leetcode.net/codec.html#:~:text=very%20close%20to%20the%20entropy,use%20the%20previous%20frame%20again), while FFV1’s status as a top archival codec is noted by preservation experts[en.wikipedia.org](https://en.wikipedia.org/wiki/FFV1#:~:text=FFV1%20,4)[en.wikipedia.org](https://en.wikipedia.org/wiki/FFV1#:~:text=FFV1%20proved%20to%20be%20a,14). VP9 and other codec settings are drawn from FFmpeg’s official usage guides[ffmpeg.org](https://ffmpeg.org/ffmpeg-codecs.html#:~:text=screen) and community Q&As[superuser.com](https://superuser.com/questions/1429256/producing-lossless-video-from-set-of-png-images-using-ffmpeg#:~:text=You%27re%20using%20a%20generic%20value,or%20libx265%29%20instead). All ensure that the techniques suggested are grounded in proven results and authoritative sources.

![](https://www.google.com/s2/favicons?domain=https://stackoverflow.com&sz=32)

![](https://www.google.com/s2/favicons?domain=https://en.wikipedia.org&sz=32)

![](https://www.google.com/s2/favicons?domain=https://multicorewareinc.com&sz=32)

![](https://www.google.com/s2/favicons?domain=https://superuser.com&sz=32)

![](https://www.google.com/s2/favicons?domain=https://www.reddit.com&sz=32)

Sources