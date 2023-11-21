Aurio: Audio Fingerprinting & Retrieval for .NET
================================================

Aurio is a .NET library that focuses on audio processing, analysis, media synchronization and media retrieval and implements various audio fingerprinting methods. It has been developed for research purposes and is a by-product of the media synchronization application [AudioAlign](https://github.com/protyposis/AudioAlign).


Features
--------

* 32-bit floating point audio processing engine
* File I/O through NAudio and FFmpeg
* Audio playback through NAudio
* FFT/iFFT through PFFFT, FFTW (optional) and Exocortex.DSP (optional)
* Resampling through Soxr and SecretRabbitCode/libsamplerate (optional)
* Stream windowing and overlap-adding
* STFT, inverse STFT
* Chroma
* Dynamic Time Warping
* On-line Time Warping (Dixon, Simon. "Live tracking of musical performances using on-line time warping." Proceedings of the 8th International Conference on Digital Audio Effects. 2005.)
* Fingerprinting
 *  Haitsma, Jaap, and Ton Kalker. "A highly robust audio fingerprinting system." ISMIR. 2002.
 *  Wang, Avery. "An Industrial Strength Audio Search Algorithm." ISMIR. 2003.
 *  [Echoprint](http://echoprint.me/codegen) (Ellis, Daniel PW, Brian Whitman, and Alastair Porter. "Echoprint: An open music identification service." ISMIR. 2011.)
 *  AcoustID [Chromaprint](https://acoustid.org/chromaprint)

All audio processing (incl. fingerprinting) is stream-based and supports processing of arbitrarily long streams at constant memory usage. All fingerprinting methods are implemented from scratch, not ports from existing libraries, while keeping compatibility where possible.

Aurio.WaveControls provides WPF widgets for user interfaces:

* Spectrogram / Chromagram View
* Spectrum / Graph View
* VU Meter
* Correlometer
* Time Scale
* Wave View


What's new
----------

See [CHANGELOG](CHANGELOG.md).


Support
-------

For questions and issues, please open an issue on the issue tracker. Commercial support, development
and consultation is available through [Protyposis Multimedia Solutions](https://protyposis.com).

Requirements
------------

* Windows
  - Visual Studio 2022 (with CMake tools)
* Linux
  - Ubuntu 22.04
  - CMake
  - Ninja
* .NET SDK 6.0


Build Instructions
------------------

### Windows
1. Install build environment (see requirements above)
2. Install dependencies
   - Run `install-deps.ps1` in PowerShell
3. Build native code in `cmd` (or open `.\nativesrc` project in VS 2022)
   - `"C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsall.bat" x64`
   - `cmake nativesrc --preset x64-debug`
   - `cmake --build nativesrc\out\build\x64-debug`
4. Build managed code (or open `.\src` in VS 2022)
   - `dotnet build src -c Debug`

### Linux
1. Install build environment
   - `apt install cmake ninja-build dotnet-sdk-6.0`
2. Install dependencies
   - Run `install-deps.sh`
3. Build native code
   - `cmake nativesrc --preset linux-debug`
   - `cmake --build nativesrc/out/build/linux-debug`
4. Build managed code
   - `dotnet build src -c LinuxDebug`


Documentation
-------------

Not available yet. If you have any questions, feel free to open an issue!


Publications
------------

> Mario Guggenberger. 2015. [Aurio: Audio Processing, Analysis and Retrieval](http://protyposis.net/publications/). In Proceedings of the 23rd ACM international conference on Multimedia (MM '15). ACM, New York, NY, USA, 705-708. DOI=http://dx.doi.org/10.1145/2733373.2807408


Examples
--------

### Reading, Processing & Writing

```csharp
/* Read a high definition MKV video file with FFmpeg,
 * convert it to telephone sound quality,
 * and write it to a WAV file with NAudio. */
var sourceStream = new FFmpegSourceStream(new FileInfo("high-definition-video.mkv"));
var ieee32BitStream = new IeeeStream(sourceStream);
var monoStream = new MonoStream(ieee32BitStream);
var resamplingStream = new ResamplingStream(monoStream, ResamplingQuality.Low, 8000);
var sinkStream = new NAudioSinkStream(resamplingStream);
WaveFileWriter.CreateWaveFile("telephone-audio.wav", sinkStream);
```

### Short-time Fourier Transform

```csharp
// Setup STFT with a window size of 100ms and an overlap of 50ms
var source = AudioStreamFactory.FromFileInfoIeee32(new FileInfo("somefilecontainingaudio.ext"));
var windowSize = source.Properties.SampleRate/10;
var hopSize = windowSize/2;
var stft = new STFT(source, windowSize, hopSize, WindowType.Hann);
var spectrum = new float[windowSize/2];

// Read all frames and get their spectrum
while (stft.HasNext()) {
    stft.ReadFrame(spectrum);
    // do something with the spectrum (e.g. build spectrogram)
}
```

### Generate fingerprints

```csharp
// Setup the source (AudioTrack is Aurio's internal representation of an audio file)
var audioTrack = new AudioTrack(new FileInfo("somefilecontainingaudio.ext"));

// Setup the fingerprint generator (each fingerprinting algorithms has its own namespace but works the same)
var defaultProfile = FingerprintGenerator.GetProfiles()[0]; // the first one is always the default profile
var generator = new FingerprintGenerator(defaultProfile);

// Setup the generator event listener
generator.SubFingerprintsGenerated += (sender, e) => {
    // Print the hashes
    e.SubFingerprints.ForEach(sfp => Console.WriteLine("{0,10}: {1}", sfp.Index, sfp.Hash));
};

// Generate fingerprints for the whole track
generator.Generate(audioTrack);
```

### Fingerprinting & Matching

```csharp
// Setup the sources
var audioTrack1 = new AudioTrack(new FileInfo("somefilecontainingaudio1.ext"));
var audioTrack2 = new AudioTrack(new FileInfo("somefilecontainingaudio2.ext"));

// Setup the fingerprint generator
var defaultProfile = FingerprintGenerator.GetProfiles()[0];
var generator = new FingerprintGenerator(defaultProfile);

// Create a fingerprint store
var store = new FingerprintStore(defaultProfile);

// Setup the generator event listener (a subfingerprint is a hash with its temporal index)
generator.SubFingerprintsGenerated += (sender, e) => {
    var progress = (double)e.Index / e.Indices;
    var hashes = e.SubFingerprints.Select(sfp => sfp.Hash);
    store.Add(e);
};

// Generate fingerprints for both tracks
generator.Generate(audioTrack1);
generator.Generate(audioTrack2);

// Check if tracks match
if (store.FindAllMatches().Count > 0) {
   Console.WriteLine("overlap detected!");
}
```

### Multitrack audio playback

```csharp
var drumTrack = new AudioTrack(new FileInfo("drums.wav"));
var guitarTrack = new AudioTrack(new FileInfo("guitar.wav"));
var vocalTrack = new AudioTrack(new FileInfo("vocals.wav"));
var band = new TrackList<AudioTrack>(new[] {drumTrack, guitarTrack, vocalTrack});

new MultitrackPlayer(band).Play();
```

Example Applications
--------------------

Aurio comes with a few tools and test applications that can be taken as a reference:

* Tests
  * **Aurio.FFmpeg.Test** decodes audio to wav files or video to frame images
  * **Aurio.Test.FingerprintingBenchmark** runs a file through all fingerprinting algorithms and measures the required time.
  * **Aurio.Test.FingerprintingHaitsmaKalker2002** fingerprints files and builds a hash store to match the fingerprinted files. The matches can be browsed and fingerprints inspected.
  * **Aurio.Test.FingerprintingWang2003** display the spectrogram and constellation map while fingerprinting a file.
  * **Aurio.Test.MultitrackPlayback** is a multitrack audio player with a simple user interface.
  * **Aurio.Test.RealtimeFingerprinting** fingerprints a realtime live audio stream.
  * **Aurio.Test.ResamlingStream** is a test bed for dynamic resampling.
  * **Aurio.Test.WaveViewControl** is a test bed for the WaveView WPF control.
* Tools
  * **BatchResampler** resamples wave files according to a configuration file.
  * **MusicDetector** analyzes audio files for music content.
  * **WaveCutter** cuts a number of concatenated files into slices of random length.

Aurio has originally been developed for [AudioAlign](https://github.com/protyposis/AudioAlign), a tool to automatically synchronize overlapping audio and video recordings, which uses almost all functionality of Aurio. Its sources are also available and can be used as a implementation reference.


Patents
-------

The fingerprinting methods by Haitsma&Kalker and Wang are protected worldwide, and Echoprint is protected in the US by patents. Their usage is therefore severely limited. In Europe, patented methods can be used privately or for research purposes.


License
-------

Copyright (C) 2010-2017 Mario Guggenberger <mg@protyposis.net>.
This project is released under the terms of the GNU Affero General Public License. See `LICENSE` for details. The library can be built to be free of any copyleft requirements; get in touch if the AGPL does not suit your needs.
