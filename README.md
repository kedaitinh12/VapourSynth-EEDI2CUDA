# VapourSynth-EEDI2CUDA

EEDI2 filter using CUDA

Ported from [HolyWu's plugin](https://github.com/HomeOfVapourSynthEvolution/VapourSynth-EEDI2).

## Usage

### Doubling Height
`eedi2cuda.EEDI2(clip clip, int field[, int mthresh=10, int lthresh=20, int vthresh=20, int estr=2, int dstr=4, int maxd=24, int map=0, int nt=50, int pp=1, int[] planes=[0, 1, 2], int num_streams=1, int device_id=-1])`

Arguments have the exactly same meanings with the ones of [Holy's EEDI2](https://github.com/HomeOfVapourSynthEvolution/VapourSynth-EEDI2). See descriptions there.
Only `pp=1` is implemented.

Additional arguments:
- `planes`: specify which planes to process. For `EEDI2` and `Enlarge2`, unprocessed planes in the result clip contain garbage. For `AA2`, unprocessed planes are copied as-is. 
- `num_streams`: specify the number of CUDA streams. The value must less than or equal to `core.num_threads`. A larger value increases the concurrency and also increases the GPU memory usage. The default value `num_streams=1` is already fast enough.
- `device_id`: set the GPU device ID to use. You *must* specify this argument for each call if you have multiple GPUs.

### Enlarge 2X
`eedi2cuda.Enlarge2(clip clip[, int mthresh=10, int lthresh=20, int vthresh=20, int estr=2, int dstr=4, int maxd=24, int map=0, int nt=50, int pp=1, int[] planes=[0, 1, 2], int num_streams=1, int device_id=-1])`

Enlarge the clip by 2. Offsets caused by doubling are neutralized. This can be considered as a faster equivalent of the code below:
```python3
el = core.eedi2cuda.EEDI2(clip, 1)
el = core.fmtc.resample(el, sy=[-.5, -.5 * (1 << clip.format.subsampling_h)])
el = core.std.Transpose(el)
el = core.eedi2cuda.EEDI2(el, 1)
el = core.fmtc.resample(el, sy=[-.5, -.5 * (1 << clip.format.subsampling_w)])
el = core.std.Transpose(el)
```

### Anti-aliasing
`eedi2cuda.AA2(clip clip[, int mthresh=10, int lthresh=20, int vthresh=20, int estr=2, int dstr=4, int maxd=24, int map=0, int nt=50, int pp=1, int[] planes=[0, 1, 2], int num_streams=1, int device_id=-1])`

Double and then scale back to do anti-aliasing. Offsets caused by doubling are neutralized. This can be considered as a faster equivalent of the code below:
```python3
w = clip.width
h = clip.height
aa = core.eedi2cuda.EEDI2(clip, 1)
aa = core.fmtc.resample(aa, w, h, sy=[-.5, -.5 * (1 << clip.format.subsampling_h)])
aa = core.std.Transpose(aa)
aa = core.eedi2cuda.EEDI2(aa, 1)
aa = core.fmtc.resample(aa, h, w, sy=[-.5, -.5 * (1 << clip.format.subsampling_w)])
aa = core.std.Transpose(aa)
```

It uses spline36 with [extended filter size](https://mpv.io/manual/stable/#options-correct-downscaling) for downscaling.

### Using in [mpv](https://mpv.io/)
You can use eedi2cuda in mpv as the upscaler for realtime playback.
First ensure the VapourSynth video filter is available in mpv.
Then copy the script below and save it as `eedi2enlarge.vpy`:
```python3
src16 = video_in.resize.Point(format=video_in.format.replace(bits_per_sample=16))
src16.eedi2cuda.Enlarge2().set_output()
```

In commandline option, load the script:
```cmd
mpv --vf=vapoursynth=eedi2enlarge.vpy
```

Or you can specify it in `mpv.conf`.

In most scenarios, it is not necessary to use an advanced upscaler for UV planes.
The script below does per-plane upscaling:
```python3
import vapoursynth as vs
from vapoursynth import core

src16 = video_in.resize.Point(format=video_in.format.replace(bits_per_sample=16))
w = src16.width
h = src16.height
y = core.std.ShufflePlanes(src16, 0, vs.GRAY)
y = core.eedi2cuda.Enlarge2(y)
uv = core.resize.Point(src16, w * 2, h * 2,
                       format=vs.YUV444P16,
                       resample_filter_uv="spline36")
core.std.ShufflePlanes([y, uv], [0, 1, 2], vs.YUV).set_output()
```

## Requirements
- A CUDA-enabled GPU of compute capability 5.0 or higher (Maxwell+).
- GPU driver **461.33** or newer.

## Compilation
Please refer to [build.yml](https://github.com/AmusementClub/VapourSynth-EEDI2CUDA/blob/main/.github/workflows/build.yml).

## Performance

|        |EEDI2CUDA   |Holy's EEDI2|
|--------|------------|------------|
|EEDI2   |175.33 fps  |18.44 fps   |
|Enlarge2|71.78 fps   |6.42 fps    |
|AA2     |104.56 fps  |9.62 fps    |

1080P YUV420P16 input. `num_streams=16`.
Tested on i7-10875H and RTX 2070.
