
# **Optimizing Storage of PNG Sequences: A Technical Analysis of Lossless Video Codecs**

## **Executive Summary and Primary Recommendation**

### **Overview of the Challenge**

The task of converting a series of Portable Network Graphics (PNG) screenshots into a video format for optimized storage represents a specialized challenge in digital archiving. The primary objective is to achieve the highest possible compression efficiency while maintaining perfect, bit-for-bit data fidelity. The unique characteristics of this source material—namely, desktop application captures with large static regions and high frame-to-frame similarity—allow for compression strategies that diverge significantly from those used for standard photographic or cinematic video. This report provides a detailed analysis of the most suitable lossless video codecs for this purpose, focusing on storage efficiency above all other metrics.

### **Primary Recommendation: AV1 (libaom-av1)**

For the stated primary goal of **maximum storage efficiency**, the unequivocal recommendation is the AV1 codec, specifically implemented via the libaom-av1 encoder within the FFmpeg framework. AV1 represents the current state-of-the-art in video compression, and its sophisticated inter-frame prediction capabilities are uniquely suited to the high temporal redundancy of screen capture content. Empirical data indicates that AV1, when used in its lossless mode, can yield file sizes dramatically smaller than any other lossless option, with potential storage savings exceeding 90% compared to traditional archival codecs.1 This recommendation acknowledges the significant trade-off in encoding time, which is substantially longer than other codecs; however, this aligns with the user's de-prioritization of encoding speed.

### **Secondary Recommendation: H.264 (libx264rgb)**

As a highly viable and more mature alternative, the H.264 codec, specifically using the libx264rgb encoder, is recommended. This option provides a superior balance of excellent compression efficiency and significantly faster encoding speeds compared to AV1.2 A key advantage of libx264rgb is its ability to operate directly in the RGB color space of the source PNG files, eliminating any potential for data loss or complexity arising from color space conversions. It stands as a robust, practical, and highly efficient choice.

### **Key Takeaway**

The analysis conclusively demonstrates that modern, inter-frame video codecs operating in a mathematically lossless mode are vastly superior to traditional intra-frame archival codecs (such as FFV1) for the specific use case of compressing screen recordings. The ability of these modern codecs to exploit redundancy between frames is the critical factor that enables extraordinary gains in storage efficiency.

## **Foundational Principles for Lossless Screen Content Archiving**

### **The Unique Characteristics of Screen Capture Content**

The selection of an optimal codec is fundamentally dictated by the nature of the source material. Desktop screen recordings possess distinct characteristics that set them apart from natural, camera-captured video. They are typically composed of large areas of solid, uniform color, sharp vector-rendered text, and simple gradients. Critically, the changes between consecutive frames are often minimal and highly localized, such as the movement of a mouse pointer, a blinking text cursor, or the appearance of newly typed characters.

This property, known as high temporal coherence or redundancy, is the single most important factor for compression.4 It means that the vast majority of pixel data in one frame is identical to the next. Compression algorithms designed to identify and eliminate this temporal redundancy will therefore perform exceptionally well. This is why variable bit rate (VBR) or constant quality (CRF) modes in modern codecs are so effective for non-camera video; they can allocate very few bits to unchanged portions of the screen, resulting in massive data savings.5

### **The Imperative of Mathematical Losslessness**

A crucial distinction must be made between codecs that are "visually lossless" and those that are "mathematically lossless." Formats often used in professional video editing, such as Apple ProRes and Avid DNxHD, are frequently described as "virtually" or "visually" lossless. However, they are fundamentally lossy codecs based on the Discrete Cosine Transform (DCT), similar to JPEG, and will always introduce some degree of data loss upon transcoding.6 For true archival purposes where perfect data integrity is required, only mathematically lossless compression is acceptable. Achieving this with modern video codecs requires overcoming two primary hurdles.

#### **Hurdle 1: Quantization**

Quantization is the process of reducing the precision of data, and it is the primary mechanism through which lossy compression operates. To achieve mathematical losslessness, this step must be completely disabled. In the FFmpeg implementations of modern codecs, this is typically accomplished with specific flags:

* For H.264 (libx264), setting the Quantization Parameter to zero (-qp 0\) is the recommended method. A Constant Rate Factor of zero (-crf 0\) also enables lossless mode for certain profiles.7  
* For HEVC (libx265), the parameter \-x265-params lossless=1 is used to activate the lossless pathway.9  
* For AV1 (libaom-av1), setting \-crf 0 enables its lossless mode.11

When these modes are active, the codecs are mathematically lossless, a fact that can be independently verified using frame-by-frame checksums or structural similarity (SSIM) metrics, which will report perfect fidelity.8

#### **Hurdle 2: Color Space Conversion**

Perhaps the most critical and easily overlooked pitfall in creating a lossless video from PNG files is the handling of color space. PNG images are inherently encoded in the RGB color space.13 Most video codecs, by contrast, are optimized for and default to a YUV color space. To improve compression efficiency, video is commonly subjected to chroma subsampling, where the color (chrominance) information is stored at a lower resolution than the brightness (luma) information. A common format like yuv420p discards 75% of the original color data.

This conversion from full-resolution RGB to subsampled YUV is a lossy process that occurs *before* the video stream is even compressed. Therefore, a command that appears to be lossless (e.g., using \-crf 0\) will silently fail the user's objective if it allows this default color conversion to occur.14 To ensure a truly lossless pipeline from a PNG source, one of two approaches must be taken:

1. Utilize a codec variant that operates natively in the RGB color space, such as libx264rgb.2  
2. Explicitly instruct the encoder to use a non-subsampled pixel format, such as yuv444p, which maintains full color resolution throughout the conversion and encoding process. This format is supported by libx264, libx265, and libaom-av1.18

### **Inter-frame vs. Intra-frame: The Key to Efficiency**

The dramatic difference in performance between codec families stems from their fundamental compression paradigm.

* **Intra-frame Codecs:** These codecs, including FFV1, HuffYUV, and Motion JPEG, compress each video frame independently of all others.13 This approach can be conceptualized as creating a sequence of individually compressed images (like a series of ZIP files). The primary advantages of this method are rapid decoding and exceptional error resilience; damage to one frame cannot propagate and corrupt subsequent frames. This robustness is why intra-frame codecs are the preferred choice for professional video editing intermediates and long-term institutional archives where format simplicity and data integrity are paramount.22 However, by design, they cannot exploit the vast temporal redundancy present in screen recordings.  
* **Inter-frame Codecs:** These codecs, including H.264, HEVC, and AV1, were designed for efficient video distribution and streaming. They operate on a Group of Pictures (GOP) and use a combination of frame types.25 An I-frame (or keyframe) is a fully encoded picture, similar to an intra-frame. However, subsequent P-frames and B-frames store only the *differences* from previous or surrounding frames. For a static screen recording, these differences are minimal, allowing P- and B-frames to be encoded with exceptionally few bits. It is this inter-frame prediction that is the core mechanism behind their superior compression efficiency for this specific type of content.1

## **A Comparative Analysis of Viable Lossless Codecs**

### **The Archival Standard: FFV1 (FF Video 1\)**

FFV1 is an open-source, patent-unencumbered, lossless intra-frame codec developed within the FFmpeg project and formally standardized as RFC 9043\.21 It has gained widespread adoption in the archival community, with institutions like the U.S. Library of Congress endorsing it as a suitable preservation format.22

Its strengths lie in its design for long-term preservation. It features fast encoding and decoding, support for a wide array of pixel formats and bit depths, and robust error detection and resilience capabilities, particularly when configured to encode each frame independently (-g 1\) and include per-slice checksums (-slicecrc 1).12

However, its primary weakness for this specific use case is a direct consequence of its intra-frame nature. Because it does not employ sophisticated motion prediction, its compression efficiency on highly static screen content is significantly lower than modern inter-frame codecs.1 While it can adapt its context model across multiple frames (-g \> 1), the resulting compression gains are minor compared to true inter-frame prediction.21

**Verdict:** FFV1 is an outstanding codec for general-purpose digital preservation and is the correct choice when error resilience and format simplicity are the highest priorities. For the user's primary goal of maximum storage efficiency for screen captures, it is not the optimal choice.

### **The High-Efficiency Contenders: Modern Codecs in Lossless Mode**

The analysis now turns to the codecs designed for distribution, but operated in their mathematically lossless modes.

#### **H.264 (libx264 / libx264rgb)**

As the most widely deployed video codec globally, the libx264 implementation in FFmpeg is exceptionally mature and optimized. It provides a true lossless encoding mode via the \-qp 0 parameter.7 Its most compelling feature for this task is the libx264rgb variant, an encoder that works directly in the RGB color space. This provides a perfect, one-to-one pathway from the source PNG files, eliminating any concerns about color conversion fidelity.2 In terms of performance, libx264rgb offers a superb balance, achieving high compression ratios with more moderate encoding times than its successors. Utilizing a slower preset, such as \-preset veryslow, instructs the encoder to perform a more exhaustive analysis, which trades longer computation time for improved compression efficiency—a worthwhile trade-off given the user's priorities.7

#### **HEVC / H.265 (libx265)**

HEVC is the designated successor to H.264, engineered to provide greater compression efficiency, particularly at higher resolutions.9 Its lossless mode is enabled in FFmpeg's libx265 encoder with the \-x265-params lossless=1 flag.9 However, for the specific case of 8-bit lossless RGB content, comparative tests show that HEVC's performance is not consistently superior to libx264rgb; the two codecs often trade blows in terms of final file size.2 This lack of a clear advantage, combined with generally slower encoding speeds than libx264, makes it a viable but less compelling option than H.264 for this task.

#### **AV1 (libaom-av1)**

AV1 is the current state-of-the-art, an open and royalty-free codec developed by the Alliance for Open Media to surpass HEVC in efficiency.19 User-provided benchmarks confirm that this efficiency advantage holds true in lossless mode, with AV1 producing the smallest file sizes by a significant margin compared to both H.264 and HEVC.1 This superior performance comes at the cost of extremely slow encoding speeds. This trade-off can be managed via the \-cpu-used parameter, where lower values (e.g., 0 or 1\) yield the best compression at the slowest speed, and higher values (up to 8\) accelerate the process at the expense of some efficiency.11 The libaom-av1 encoder fully supports non-subsampled pixel formats like yuv444p and even planar RGB (gbrp), making it perfectly capable of a lossless conversion from PNG sources.19

**Verdict:** AV1 is the top recommendation for the user's primary requirement of maximum storage efficiency, provided the very long encoding times are an acceptable trade-off.

### **Performance and Suitability Matrix**

The following table summarizes the comparative analysis of the leading codecs against the user's specified criteria.

| Feature | FFV1 | H.264 (libx264rgb) | HEVC (libx265) | AV1 (libaom-av1) |
| :---- | :---- | :---- | :---- | :---- |
| **Compression Efficiency** | Baseline (Good) | Excellent | Excellent | **State-of-the-Art** |
| **Encoding Speed** | Fast | Moderate | Slow | **Very Slow** |
| **Maturity / Stability** | Very High | Very High | High | High |
| **Primary Use Case** | Archival Safety, Editing | Balanced Efficiency & Speed | High-Resolution Archiving | **Maximum Storage Optimization** |

This comparison reveals a clear evolutionary trend: newer, more algorithmically complex codecs consistently deliver better compression at the cost of increased computational demand. The user's choice is therefore not simply between codecs, but where to position themselves on this efficiency-versus-time curve.

## **Implementation Guide: FFmpeg Command-Line Recipes**

This section provides practical, copy-and-paste ready FFmpeg commands for encoding a sequence of PNG files. Each parameter is explained in detail to ensure a full understanding of the process.

### **Base Command for PNG Sequence Input**

All recipes will use a similar input structure. The following flags are fundamental for handling sequentially numbered images:

* \-framerate \<rate\>: This input option sets the frame rate of the resulting video. For screen captures, a lower rate like 10 or 15 is often sufficient and appropriate.16  
* \-start\_number \<n\>: Use this option if the image sequence does not begin with the number 0 or 1\.16  
* \-i "capture.%04d.png": This specifies the input file pattern. The %d is a format specifier that is replaced by the frame number. The 04 indicates that the number is zero-padded to four digits (e.g., capture.0001.png, capture.0002.png).29

It is important to note that for inter-frame codecs like H.264 and AV1, the keyframe interval (-g option) should generally not be manually set to a low value. The encoder's default behavior, which places keyframes at scene changes or at a very large maximum interval, is optimal for compressing static content. Forcing frequent keyframes would severely degrade compression efficiency, defeating the primary purpose of using these codecs.

### **Recipe 1: H.264 (libx264rgb) \- The Balanced Approach**

This command provides an excellent balance of high compression and reasonable encoding time, using a direct RGB pathway.

Bash

ffmpeg \-framerate 15 \-i "capture.%04d.png" \-c:v libx264rgb \-qp 0 \-preset veryslow \-pix\_fmt rgb24 output\_h264.mkv

**Parameter Breakdown:**

* \-c:v libx264rgb: Selects the dedicated H.264 RGB encoder, the most direct and foolproof method for handling RGB source data from PNGs.2  
* \-qp 0: Sets the Quantization Parameter to 0\. This is the recommended method for enabling mathematically lossless mode in libx264.7  
* \-preset veryslow: Instructs the encoder to use its most computationally intensive and exhaustive analysis settings. This significantly increases encoding time but yields the best possible compression for a given quality level.7  
* \-pix\_fmt rgb24: Explicitly specifies the 24-bit RGB pixel format for the output stream, ensuring no unintended color space conversions occur.17

### **Recipe 2: AV1 (libaom-av1) \- The Maximum Efficiency Approach**

This command is optimized for the smallest possible file size, at the cost of a very long encoding process.

Bash

ffmpeg \-framerate 15 \-i "capture.%04d.png" \-c:v libaom-av1 \-crf 0 \-cpu-used 0 \-pix\_fmt yuv444p output\_av1.mkv

**Parameter Breakdown:**

* \-c:v libaom-av1: Selects the reference AV1 encoder from the Alliance for Open Media.19  
* \-crf 0: Sets the Constant Rate Factor to 0, which activates the lossless encoding mode for libaom-av1.11  
* \-cpu-used 0: This is the most critical performance-tuning parameter. A value of 0 results in the slowest encoding process but achieves the highest compression efficiency. This value can be increased (up to 8\) to substantially reduce encoding time, with a corresponding increase in the final file size.11  
* \-pix\_fmt yuv444p: Since libaom-av1 does not have a dedicated RGB mode, this parameter is essential. It forces the use of the YUV 4:4:4 pixel format, which performs the conversion from the source RGB without any chroma subsampling, thus preserving all color information losslessly.19

### **Recipe 3: FFV1 \- The Archival Approach (for Completeness)**

This command demonstrates the best practices for creating a robust, archival-grade file using FFV1, prioritizing error resilience over compression.

Bash

ffmpeg \-framerate 15 \-i "capture.%04d.png" \-c:v ffv1 \-level 3 \-coder 1 \-context 1 \-g 1 \-slicecrc 1 output\_ffv1.mkv

**Parameter Breakdown:**

* \-c:v ffv1: Selects the FFV1 codec.21  
* \-level 3: Specifies version 3 of the FFV1 bitstream, which supports modern features like multithreading and slice-based checksums.21  
* \-coder 1: Selects the Range Coder for entropy coding, which typically offers slightly better compression than the default Golomb-Rice coder.21  
* \-context 1: Uses the larger context model, which can also improve compression.21  
* \-g 1: Sets the Group of Pictures (GOP) size to 1\. This forces every frame to be an intra-frame, maximizing error resilience as damage to one frame cannot affect any others. This is a cornerstone of archival best practice for FFV1.21  
* \-slicecrc 1: Enables the writing of a CRC checksum for each slice within a frame. This allows a decoder to robustly detect any corruption that may occur during storage or transmission, a key feature for long-term preservation.21

## **Final Recommendations and Verification**

### **Concluding Recommendation**

The optimal choice of codec depends on the user's tolerance for encoding time.

* For **absolute maximum storage efficiency**, where encoding time is not a constraint, **AV1 (libaom-av1)** is the superior choice.  
* For a **more practical balance of excellent compression and faster encoding**, **H.264 (libx264rgb)** is the recommended alternative.

Both options will deliver mathematically lossless results and significantly outperform traditional intra-frame codecs for this specific content type.

### **Verification of Losslessness**

To provide full confidence in the archival process, it is essential to verify that the output video is a mathematically perfect representation of the original PNG sequence. FFmpeg provides powerful, built-in tools for this verification.

#### **Method 1: Frame Hash Comparison (framemd5)**

This method provides absolute, cryptographic proof of losslessness by comparing the MD5 hash of every original frame against every decoded frame from the output video. If the hashes match perfectly, the process was bit-for-bit identical.12

1. **Generate a hash file from the original PNG sequence:**  
   Bash  
   ffmpeg \-i "capture.%04d.png" \-f framemd5 original\_hashes.txt

2. **Generate a hash file from the newly encoded video:**  
   Bash  
   ffmpeg \-i output\_av1.mkv \-f framemd5 encoded\_hashes.txt

3. **Compare the two output files.** Any standard file comparison tool (e.g., diff on Linux/macOS, FC on Windows) can be used. If the files are identical, the encoding was perfectly lossless.

#### **Method 2: Structural Similarity Index (ssim)**

This method provides a faster check by calculating the structural similarity between the source and the encoded video during a single pass. For a mathematically lossless encode, the SSIM score will be a perfect 1.0, and the corresponding Peak Signal-to-Noise Ratio (PSNR) will be infinite (inf).8

* **Run the SSIM and PSNR comparison:**  
  Bash  
  ffmpeg \-i output\_av1.mkv \-i "capture.%04d.png" \-lavfi "\[0:v\]\[1:v\]ssim;\[0:v\]\[1:v\]psnr" \-f null \-

  The console output should report All:1.000000 (inf) for SSIM and psnr\_avg:inf for PSNR, confirming perfect fidelity.

By employing these verification techniques, the integrity of the archived screen captures can be ensured with a high degree of confidence, completing a robust and professional archival workflow.

#### **Works cited**

1. ffmpeg \- Is it okay to re-encode videos from traditional lossless ..., accessed October 28, 2025, [https://superuser.com/questions/1796921/is-it-okay-to-re-encode-videos-from-traditional-lossless-codec-to-modern-lossles](https://superuser.com/questions/1796921/is-it-okay-to-re-encode-videos-from-traditional-lossless-codec-to-modern-lossles)  
2. Best Lossless Video Codec for 8 bpc RGB image sequences? \- Super User, accessed October 28, 2025, [https://superuser.com/questions/1810349/best-lossless-video-codec-for-8-bpc-rgb-image-sequences](https://superuser.com/questions/1810349/best-lossless-video-codec-for-8-bpc-rgb-image-sequences)  
3. Best Truly Lossless Codec? : r/vfx \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/vfx/comments/16u9797/best\_truly\_lossless\_codec/](https://www.reddit.com/r/vfx/comments/16u9797/best_truly_lossless_codec/)  
4. What is the most compression efficient mathematically lossless video codec? \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/compression/comments/16z4jla/what\_is\_the\_most\_compression\_efficient/](https://www.reddit.com/r/compression/comments/16z4jla/what_is_the_most_compression_efficient/)  
5. Choosing a video codec for screen recording \- Stack Overflow, accessed October 28, 2025, [https://stackoverflow.com/questions/8937899/choosing-a-video-codec-for-screen-recording](https://stackoverflow.com/questions/8937899/choosing-a-video-codec-for-screen-recording)  
6. FFV1 vs H.264 Codec for archiving : r/videography \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/videography/comments/ga67hg/ffv1\_vs\_h264\_codec\_for\_archiving/](https://www.reddit.com/r/videography/comments/ga67hg/ffv1_vs_h264_codec_for_archiving/)  
7. Encode/H.264 \- FFmpeg Wiki, accessed October 28, 2025, [https://trac.ffmpeg.org/wiki/Encode/H.264](https://trac.ffmpeg.org/wiki/Encode/H.264)  
8. ffmpeg \- Using h264 in loseless mode brings small unexpected results, accessed October 28, 2025, [https://video.stackexchange.com/questions/16674/using-h264-in-loseless-mode-brings-small-unexpected-results](https://video.stackexchange.com/questions/16674/using-h264-in-loseless-mode-brings-small-unexpected-results)  
9. Encode/H.265 \- FFmpeg Wiki, accessed October 28, 2025, [https://trac.ffmpeg.org/wiki/Encode/H.265](https://trac.ffmpeg.org/wiki/Encode/H.265)  
10. How to correctly use x265 lossless parameter with avconv \- Super User, accessed October 28, 2025, [https://superuser.com/questions/976230/how-to-correctly-use-x265-lossless-parameter-with-avconv](https://superuser.com/questions/976230/how-to-correctly-use-x265-lossless-parameter-with-avconv)  
11. Encode/AV1 \- FFmpeg Wiki, accessed October 28, 2025, [https://trac.ffmpeg.org/wiki/Encode/AV1](https://trac.ffmpeg.org/wiki/Encode/AV1)  
12. Comparative analysis of uncompressed AVI and FFV1 video \- Digital Preservation Coalition, accessed October 28, 2025, [https://www.dpconline.org/blog/uncompressed-avi-ffv1](https://www.dpconline.org/blog/uncompressed-avi-ffv1)  
13. Lossless Video Format: 7 Popular Formats and How to Choose | Cloudinary, accessed October 28, 2025, [https://cloudinary.com/guides/video-formats/lossless-video-format-7-popular-formats-and-how-to-choose](https://cloudinary.com/guides/video-formats/lossless-video-format-7-popular-formats-and-how-to-choose)  
14. Image sequence conversion to HEVC with ffmpeg \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/ffmpeg/comments/18ldksy/image\_sequence\_conversion\_to\_hevc\_with\_ffmpeg/](https://www.reddit.com/r/ffmpeg/comments/18ldksy/image_sequence_conversion_to_hevc_with_ffmpeg/)  
15. How To Convert PNG Sequence To Video in 5 Effective Ways \- Filmora \- Wondershare, accessed October 28, 2025, [https://filmora.wondershare.com/video-editing-tips/png-sequence-to-video.html](https://filmora.wondershare.com/video-editing-tips/png-sequence-to-video.html)  
16. FFMPEG An Intermediate Guide/image sequence \- Wikibooks, open ..., accessed October 28, 2025, [https://en.wikibooks.org/wiki/FFMPEG\_An\_Intermediate\_Guide/image\_sequence](https://en.wikibooks.org/wiki/FFMPEG_An_Intermediate_Guide/image_sequence)  
17. FFMPEG RGB Lossless conversion video \- Stack Overflow, accessed October 28, 2025, [https://stackoverflow.com/questions/45929024/ffmpeg-rgb-lossless-conversion-video](https://stackoverflow.com/questions/45929024/ffmpeg-rgb-lossless-conversion-video)  
18. h.264 \- h264 lossless coding \- Stack Overflow, accessed October 28, 2025, [https://stackoverflow.com/questions/6701805/h264-lossless-coding](https://stackoverflow.com/questions/6701805/h264-lossless-coding)  
19. ORI Encoding Guidelines \- AV1 \- GitHub Pages, accessed October 28, 2025, [https://academysoftwarefoundation.github.io/EncodingGuidelines/EncodeAv1.html](https://academysoftwarefoundation.github.io/EncodingGuidelines/EncodeAv1.html)  
20. HEVC/H.265 Encoding \- GitHub Pages, accessed October 28, 2025, [https://academysoftwarefoundation.github.io/EncodingGuidelines/EncodeHevc.html](https://academysoftwarefoundation.github.io/EncodingGuidelines/EncodeHevc.html)  
21. FFV1 \- Codec Wiki, accessed October 28, 2025, [https://wiki.x266.mov/docs/video/FFV1](https://wiki.x266.mov/docs/video/FFV1)  
22. FFV1 \- Wikipedia, accessed October 28, 2025, [https://en.wikipedia.org/wiki/FFV1](https://en.wikipedia.org/wiki/FFV1)  
23. FFV1: The Gains of Lossless \- Bitstreams: The Digital Collections Blog, accessed October 28, 2025, [https://blogs.library.duke.edu/bitstreams/2021/03/12/ffv1-the-gains-of-lossless/](https://blogs.library.duke.edu/bitstreams/2021/03/12/ffv1-the-gains-of-lossless/)  
24. FFV1 vs other formats for preservation \- Google Groups, accessed October 28, 2025, [https://groups.google.com/g/archivematica/c/HulV96gJ0go/m/\_d4GH\_toc0EJ](https://groups.google.com/g/archivematica/c/HulV96gJ0go/m/_d4GH_toc0EJ)  
25. Setting the Right Keyframe Interval for Streaming \- FastPix, accessed October 28, 2025, [https://www.fastpix.io/blog/how-to-set-the-right-keyframe-interval-for-streaming](https://www.fastpix.io/blog/how-to-set-the-right-keyframe-interval-for-streaming)  
26. How to set the right Keyframe Interval for streaming? \- Gumlet, accessed October 28, 2025, [https://www.gumlet.com/learn/keyframe-interval/](https://www.gumlet.com/learn/keyframe-interval/)  
27. best compression/method for high fps screen capture of a series of abstract flicker frames and how best to choose final compression of captured footage \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/compression/comments/1luhjfs/best\_compressionmethod\_for\_high\_fps\_screen/](https://www.reddit.com/r/compression/comments/1luhjfs/best_compressionmethod_for_high_fps_screen/)  
28. How to get a lossless encoding with ffmpeg \- libx265 \- Stack Overflow, accessed October 28, 2025, [https://stackoverflow.com/questions/37344997/how-to-get-a-lossless-encoding-with-ffmpeg-libx265](https://stackoverflow.com/questions/37344997/how-to-get-a-lossless-encoding-with-ffmpeg-libx265)  
29. How to Create a Video from Images Using FFmpeg? \- Gumlet, accessed October 28, 2025, [https://www.gumlet.com/learn/ffmpeg-images-to-video/](https://www.gumlet.com/learn/ffmpeg-images-to-video/)  
30. Using FFmpeg to convert image sequences to video \- Brendan Dawes, accessed October 28, 2025, [https://brendandawes.com/blog/ffmpeg-images-to-video](https://brendandawes.com/blog/ffmpeg-images-to-video)  
31. ffmpeg \- libaom-av1 PSNR Spikes? \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/AV1/comments/1gmbzsb/ffmpeg\_libaomav1\_psnr\_spikes/](https://www.reddit.com/r/AV1/comments/1gmbzsb/ffmpeg_libaomav1_psnr_spikes/)  
32. Encode/FFV1 \- FFmpeg Wiki, accessed October 28, 2025, [https://trac.ffmpeg.org/wiki/Encode/FFV1](https://trac.ffmpeg.org/wiki/Encode/FFV1)  
33. How to extract 16-bit PNG frame from lossless x264 video \- Stack Overflow, accessed October 28, 2025, [https://stackoverflow.com/questions/44248921/how-to-extract-16-bit-png-frame-from-lossless-x264-video](https://stackoverflow.com/questions/44248921/how-to-extract-16-bit-png-frame-from-lossless-x264-video)