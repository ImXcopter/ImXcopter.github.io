# moviepy用GPU加速的方法 - cupy

首先安装cupy，安装方法自行百度！

## moviepy默认的安装路径

```
C:\Users\Administrator\AppData\Local\Programs\Python\Python312\Lib\site-packages\moviepy
```

## 1、修改 moviepy/video/tools/drawing.py

在这段代码中，我们用 cupy 的操作替换了 numpy 的操作，以便利用 GPU 加速处理。主要的修改点包括：

1. 导入 cupy 并用 cp 作为别名。
2. 使用 cp.copy 替代 numpy 的复制操作。
3. 使用 cp.dstack 替代 numpy 的堆叠操作。

请注意，cupy 的数组操作与 numpy 基本相同，但为了确保代码运行在 GPU 上，所有的数组操作都需要用 cupy 替代 numpy。另外，还需要确保 im1、im2 和 mask 这些输入也是 cupy 数组。如果它们是 numpy 数组，可以使用 cp.asarray 将它们转换为 cupy 数组。

```python
import numpy as np
import cupy as cp

def blit(im1, im2, pos=None, mask=None, ismask=False):
    """ Blit an image over another.

    Blits ``im1`` on ``im2`` as position ``pos=(x,y)``, using the
    ``mask`` if provided. If ``im1`` and ``im2`` are mask pictures
    (2D float arrays) then ``ismask`` must be ``True``.
    """
    if pos is None:
        pos = [0, 0]

    # Convert all inputs to cupy arrays if they are not already
    im1 = cp.asarray(im1)
    im2 = cp.asarray(im2)
    if mask is not None:
        mask = cp.asarray(mask)

    # xp1, yp1, xp2, yp2 = blit area on im2
    # x1, y1, x2, y2 = area of im1 to blit on im2
    xp, yp = pos
    x1 = max(0, -xp)
    y1 = max(0, -yp)
    h1, w1 = im1.shape[:2]
    h2, w2 = im2.shape[:2]
    xp2 = min(w2, xp + w1)
    yp2 = min(h2, yp + h1)
    x2 = min(w1, w2 - xp)
    y2 = min(h1, h2 - yp)
    xp1 = max(0, xp)
    yp1 = max(0, yp)

    if (xp1 >= xp2) or (yp1 >= yp2):
        return im2

    blitted = im1[y1:y2, x1:x2]

    new_im2 = cp.copy(im2)

    if mask is None:
        new_im2[yp1:yp2, xp1:xp2] = blitted
    else:
        mask = mask[y1:y2, x1:x2]
        if len(im1.shape) == 3:
            mask = cp.dstack(3 * [mask])
        blit_region = new_im2[yp1:yp2, xp1:xp2]
        new_im2[yp1:yp2, xp1:xp2] = (1.0 * mask * blitted + (1.0 - mask) * blit_region)

    return new_im2.astype('uint8') if (not ismask) else new_im2
```

替换color_gradient函数：

```python
def color_gradient(size, p1, p2=None, vector=None, r=None, col1=0, col2=1.0,
                   shape='linear', offset=0):
    w, h = size

    col1 = cp.array(col1).astype(float)
    col2 = cp.array(col2).astype(float)

    if shape == 'bilinear':
        if vector is None:
            vector = cp.array(p2) - cp.array(p1)

        m1, m2 = [ color_gradient(size, p1, vector=v, col1=1.0, col2=0,
                                  shape='linear', offset=offset)
                   for v in [vector, -vector]]

        arr = cp.maximum(m1, m2)
        if col1.size > 1:
            arr = cp.dstack(3 * [arr])
        return arr * col1 + (1 - arr) * col2

    p1 = cp.array(p1[::-1]).astype(float)

    if vector is None and p2:
        p2 = cp.array(p2[::-1])
        vector = p2 - p1
    else:
        vector = cp.array(vector[::-1])
        p2 = p1 + vector

    if vector:
        norm = cp.linalg.norm(vector)

    M = cp.dstack(cp.meshgrid(range(w), range(h))[::-1]).astype(float)

    if shape == 'linear':

        n_vec = vector / norm**2

        p1 = p1 + offset * vector
        arr = (M - p1).dot(n_vec) / (1 - offset)
        arr = cp.minimum(1, cp.maximum(0, arr))
        if col1.size > 1:
            arr = cp.dstack(3 * [arr])
        return arr * col1 + (1 - arr) * col2

    elif shape == 'radial':
        if r is None:
            r = norm

        if r == 0:
            arr = cp.ones((h, w))
        else:
            arr = (cp.sqrt(((M - p1) ** 2).sum(axis=2))) - offset * r
            arr = arr / ((1 - offset) * r)
            arr = cp.minimum(1.0, cp.maximum(0, arr))

        if col1.size > 1:
            arr = cp.dstack(3 * [arr])
        return (1 - arr) * col1 + arr * col2
```

## 2、修改 moviepy/video/compositing/CompositeVideoClip.py

要将 CompositeVideoClip.py 中的代码修改为支持 cupy 加速，我们需要将涉及 numpy 操作的部分替换为 cupy 操作。通过这些修改，CompositeVideoClip 类和 clips_array 函数可以利用 cupy 提供的 GPU 加速功能，从而提高处理视频剪辑的效率。

```python
import numpy as np
import cupy as cp

from moviepy.audio.AudioClip import CompositeAudioClip
from moviepy.video.VideoClip import ColorClip, VideoClip

# CompositeVideoClip

class CompositeVideoClip(VideoClip):

    """

    A VideoClip made of other videoclips displayed together. This is the
    base class for most compositions.

    Parameters
    ----------

    size
      The size (height x width) of the final clip.

    clips
      A list of videoclips. Each clip of the list will
      be displayed below the clips appearing after it in the list.
      For each clip:

      - The attribute ``pos`` determines where the clip is placed.
          See ``VideoClip.set_pos``
      - The mask of the clip determines which parts are visible.

      Finally, if all the clips in the list have their ``duration``
      attribute set, then the duration of the composite video clip
      is computed automatically

    bg_color
      Color for the unmasked and unfilled regions. Set to None for these
      regions to be transparent (will be slower).

    use_bgclip
      Set to True if the first clip in the list should be used as the
      'background' on which all other clips are blitted. That first clip must
      have the same size as the final clip. If it has no transparency, the final
      clip will have no mask.

    The clip with the highest FPS will be the FPS of the composite clip.

    """

    def __init__(self, clips, size=None, bg_color=None, use_bgclip=False,
                 ismask=False):

        if size is None:
            size = clips[0].size


        if use_bgclip and (clips[0].mask is None):
            transparent = False
        else:
            transparent = (bg_color is None)

        if bg_color is None:
            bg_color = 0.0 if ismask else (0, 0, 0)

        fpss = [c.fps for c in clips if getattr(c, 'fps', None)]
        self.fps = max(fpss) if fpss else None

        VideoClip.__init__(self)

        self.size = size
        self.ismask = ismask
        self.clips = clips
        self.bg_color = bg_color

        if use_bgclip:
            self.bg = clips[0]
            self.clips = clips[1:]
            self.created_bg = False
        else:
            self.clips = clips
            self.bg = ColorClip(size, color=self.bg_color)
            self.created_bg = True


        # compute duration
        ends = [c.end for c in self.clips]
        if None not in ends:
            duration = max(ends)
            self.duration = duration
            self.end = duration

        # compute audio
        audioclips = [v.audio for v in self.clips if v.audio is not None]
        if audioclips:
            self.audio = CompositeAudioClip(audioclips)

        # compute mask if necessary
        if transparent:
            maskclips = [(c.mask if (c.mask is not None) else
                          c.add_mask().mask).set_position(c.pos)
                          .set_end(c.end).set_start(c.start, change_end=False)
                          for c in self.clips]

            self.mask = CompositeVideoClip(maskclips, self.size, ismask=True,
                                           bg_color=0.0)

        def make_frame(t):
            """ The clips playing at time `t` are blitted over one
                another. """
            f = self.bg.get_frame(t)
            for c in self.playing_clips(t):
                f = c.blit_on(f, t)
            return f

        self.make_frame = make_frame

    def playing_clips(self, t=0):
        """ Returns a list of the clips in the composite clips that are
            actually playing at the given time `t`. """
        return [c for c in self.clips if c.is_playing(t)]

    def close(self):
        if self.created_bg and self.bg:
            # Only close the background clip if it was locally created.
            # Otherwise, it remains the job of whoever created it.
            self.bg.close()
            self.bg = None
        if hasattr(self, "audio") and self.audio:
            self.audio.close()
            self.audio = None


def clips_array(array, rows_widths=None, cols_widths=None, bg_color=None):

    """

    rows_widths
      widths of the different rows in pixels. If None, is set automatically.

    cols_widths
      widths of the different columns in pixels. If None, is set automatically.

    bg_color
       Fill color for the masked and unfilled regions. Set to None for these
       regions to be transparent (will be slower).

    """

    array = cp.array(array)
    sizes_array = cp.array([[c.size for c in line] for line in array])

    # find row width and col_widths automatically if not provided
    if rows_widths is None:
        rows_widths = sizes_array[:,:,1].max(axis=1)
    if cols_widths is None:
        cols_widths = sizes_array[:,:,0].max(axis=0)

    xx = cp.cumsum([0] + list(cols_widths))
    yy = cp.cumsum([0] + list(rows_widths))

    for j, (x, cw) in enumerate(zip(xx[:-1], cols_widths)):
        for i, (y, rw) in enumerate(zip(yy[:-1], rows_widths)):
            clip = array[i, j]
            w, h = clip.size
            if (w < cw) or (h < rw):
                clip = (CompositeVideoClip([clip.set_position('center')],
                                          size=(cw, rw),
                                          bg_color=bg_color).
                                     set_duration(clip.duration))

            array[i, j] = clip.set_position((x, y))

    return CompositeVideoClip(array.flatten(), size=(xx[-1], yy[-1]), bg_color=bg_color)
```

## 3、修改 moviepy/Clip.py

在 iter_frames 方法中修改以下部分：

```python
if (dtype is not None) and (frame.dtype != dtype):
    frame = cp.asnumpy(frame).astype(dtype)
```

## 4、修改 moviepy/video/VideoClip.py

导入 cupy:

在文件头部添加 cupy 的导入。

```python
import cupy as cp
```

修改 fill_array 方法:

```python
def fill_array(self, pre_array, shape=(0, 0)):
    pre_shape = pre_array.shape
    dx = shape[0] - pre_shape[0]
    dy = shape[1] - pre_shape[1]
    post_array = pre_array
    if dx < 0:
        post_array = pre_array[:shape[0]]
    elif dx > 0:
        x_1 = cp.array([[[1, 1, 1]] * pre_shape[1]] * dx)
        post_array = cp.vstack((pre_array, x_1))
    if dy < 0:
        post_array = post_array[:, :shape[1]]
    elif dy > 0:
        x_1 = cp.array([[[1, 1, 1]] * dy] * post_array.shape[0])
        post_array = cp.hstack((post_array, x_1))
    return post_array
```

修改 blit_on 方法:

```python
def blit_on(self, picture, t):
    """
    Returns the result of the blit of the clip's frame at time `t`
    on the given `picture`, the position of the clip being given
    by the clip's ``pos`` attribute. Meant for compositing.
    """
    hf, wf = framesize = picture.shape[:2]

    if self.ismask and picture.max():
        return cp.minimum(1, picture + self.blit_on(cp.zeros(framesize), t))

    ct = t - self.start  # clip time

    # GET IMAGE AND MASK IF ANY

    img = self.get_frame(ct)
    mask = self.mask.get_frame(ct) if self.mask else None

    if mask is not None and ((img.shape[0] != mask.shape[0]) or (img.shape[1] != mask.shape[1])):
        img = self.fill_array(img, mask.shape)

    hi, wi = img.shape[:2]

    # SET POSITION
    pos = self.pos(ct)

    # preprocess short writings of the position
    if isinstance(pos, str):
        pos = {'center': ['center', 'center'],
               'left': ['left', 'center'],
               'right': ['right', 'center'],
               'top': ['center', 'top'],
               'bottom': ['center', 'bottom']}[pos]
    else:
        pos = list(pos)

    # is the position relative (given in % of the clip's size) ?
    if self.relative_pos:
        for i, dim in enumerate([wf, hf]):
            if not isinstance(pos[i], str):
                pos[i] = dim * pos[i]

    if isinstance(pos[0], str):
        D = {'left': 0, 'center': (wf - wi) / 2, 'right': wf - wi}
        pos[0] = D[pos[0]]

    if isinstance(pos[1], str):
        D = {'top': 0, 'center': (hf - hi) / 2, 'bottom': hf - hi}
        pos[1] = D[pos[1]]

    pos = map(int, pos)

    return blit(img, picture, pos, mask=mask, ismask=self.ismask)
```

修改 to_RGB 方法:

```python
def to_RGB(self):
    """Return a non-mask video clip made from the mask video clip."""
    if self.ismask:
        f = lambda pic: cp.dstack(3 * [255 * pic]).astype('uint8')
        newclip = self.fl_image(f)
        newclip.ismask = False
        return newclip
    else:
        return self
```

修改 ImageClip 的 __init__ 方法:

```python
class ImageClip(VideoClip):
    """Class for non-moving VideoClips.

    A video clip originating from a picture. This clip will simply
    display the given picture at all times.

    Examples
    ---------

    >>> clip = ImageClip("myHouse.jpeg")
    >>> clip = ImageClip( someArray ) # a Numpy array represent

    Parameters
    -----------

    img
      Any picture file (png, tiff, jpeg, etc.) or any array representing
      an RGB image (for instance a frame from a VideoClip).

    ismask
      Set this parameter to `True` if the clip is a mask.

    transparent
      Set this parameter to `True` (default) if you want the alpha layer
      of the picture (if it exists) to be used as a mask.

    Attributes
    -----------

    img
      Array representing the image of the clip.

    """

    def __init__(self, img, ismask=False, transparent=True,
                 fromalpha=False, duration=None):
        VideoClip.__init__(self, ismask=ismask, duration=duration)

        if isinstance(img, string_types):
            img = imread(img)

        if len(img.shape) == 3:  # img is (now) a RGB(a) numpy array

            if img.shape[2] == 4:
                if fromalpha:
                    img = 1.0 * img[:, :, 3] / 255
                elif ismask:
                    img = 1.0 * img[:, :, 0] / 255
                elif transparent:
                    self.mask = ImageClip(
                        1.0 * img[:, :, 3] / 255, ismask=True)
                    img = img[:, :, :3]
            elif ismask:
                img = 1.0 * img[:, :, 0] / 255

        # if the image was just a 2D mask, it should arrive here
        # unchanged
        self.make_frame = lambda t: img
        self.size = img.shape[:2][::-1]
        self.img = img
```

修改 ColorClip 的 __init__ 方法:

```python
class ColorClip(ImageClip):
    """An ImageClip showing just one color.

    Parameters
    -----------

    size
      Size (width, height) in pixels of the clip.

    color
      If argument ``ismask`` is False, ``color`` indicates
      the color in RGB of the clip (default is black). If `ismask``
      is True, ``color`` must be  a float between 0 and 1 (default is 1)

    ismask
      Set to true if the clip will be used as a mask.

    col
      Has been deprecated. Do not use.
    """

    def __init__(self, size, color=None, ismask=False, duration=None, col=None):
        if col is not None:
            warnings.warn("The `ColorClip` parameter `col` has been deprecated."
                          " Please use `color` instead.", DeprecationWarning)
            if color is not None:
                warnings.warn("The arguments `color` and `col` have both been "
                              "passed to `ColorClip` so `col` has been ignored.",
                              UserWarning)
            else:
                color = col
        w, h = size
        shape = (h, w) if np.isscalar(color) else (h, w, len(color))
        ImageClip.__init__(self, cp.tile(color, w * h).reshape(shape),
                           ismask=ismask, duration=duration)
```

修改 TextClip 的 __init__ 方法:

```python
class TextClip(ImageClip):
    """Class for autogenerated text clips.

    Creates an ImageClip originating from a script-generated text image.
    Requires ImageMagick.

    Parameters
    -----------

    txt
      A string of the text to write. Can be replaced by argument
      ``filename``.

    filename
      The name of a file in which there is the text to write.
      Can be provided instead of argument ``txt``

    size
      Size of the picture in pixels. Can be auto-set if
      method='label', but mandatory if method='caption'.
      the height can be None, it will then be auto-determined.

    bg_color
      Color of the background. See ``TextClip.list('color')``
      for a list of acceptable names.

    color
      Color of the text. See ``TextClip.list('color')`` for a
      list of acceptable colors.

    font
      Name of the font to use. See ``TextClip.list('font')`` for
      the list of fonts you can use on your computer.

    stroke_color
      Color of the stroke (=contour line) of the text. If ``None``,
      there will be no stroke.

    stroke_width
      Width of the stroke, in pixels. Can be a float, like 1.5.

    method
      Either 'label' (default, the picture will be autosized so as to fit
      exactly the size) or 'caption' (the text will be drawn in a picture
      with fixed size provided with the ``size`` argument). If `caption`,
      the text will be wrapped automagically (sometimes it is buggy, not
      my fault, complain to the ImageMagick crew) and can be aligned or
      centered (see parameter ``align``).

    kerning
      Changes the default spacing between letters. For
      instance ``kerning=-1`` will make the letters 1 pixel nearer from
      ach other compared to the default spacing.

    align
      center | East | West | South | North . Will only work if ``method``
      is set to ``caption``

    transparent
      ``True`` (default) if you want to take into account the
      transparency in the image.

    """

    def __init__(self, txt=None, filename=None, size=None, color='black',
                 bg_color='transparent', fontsize=None, font='Courier',
                 stroke_color=None, stroke_width=1, method='label',
                 kerning=None, align='center', interline=None,
                 tempfilename=None, temptxt=None,
                 transparent=True, remove_temp=True,
                 print_cmd=False):

        if txt is not None:
            if temptxt is None:
                temptxt_fd, temptxt = tempfile.mkstemp(suffix='.txt')
                try:  # only in Python3 will this work
                    os.write(temptxt_fd, bytes(txt, 'UTF8'))
                except TypeError:  # oops, fall back to Python2
                    os.write(temptxt_fd, txt)
                os.close(temptxt_fd)
            txt = '@' + temptxt
        else:
            # use a file instead of a text.
            txt = "@%" + filename

        if size is not None:
            size = ('' if size[0] is None else str(size[0]),
                    '' if size[1] is None else str(size[1]))

        cmd = ([get_setting("IMAGEMAGICK_BINARY"),
               "-background", bg_color,
                "-fill", color,
                "-font", font])

        if fontsize is not None:
            cmd += ["-pointsize", "%d" % fontsize]
        if kerning is not None:
            cmd += ["-kerning", "%0.1f" % kerning]
        if stroke_color is not None:
            cmd += ["-stroke", stroke_color, "-strokewidth",
                    "%.01f" % stroke_width]
        if size is not None:
            cmd += ["-size", "%sx%s" % (size[0], size[1])]
        if align is not None:
            cmd += ["-gravity", align]
        if interline is not None:
            cmd += ["-interline-spacing", "%d" % interline]

        if tempfilename is None:
            tempfile_fd, tempfilename = tempfile.mkstemp(suffix='.png')
            os.close(tempfile_fd)

        cmd += ["%s:%s" % (method, txt),
                "-type", "truecolormatte", "PNG32:%s" % tempfilename]

        if print_cmd:
            print(" ".join(cmd))

        try:
            subprocess_call(cmd, logger=None)
        except (IOError, OSError) as err:
            error = ("MoviePy Error: creation of %s failed because of the "
                     "following error:\n\n%s.\n\n." % (filename, str(err))
                     + ("This error can be due to the fact that ImageMagick "
                        "is not installed on your computer, or (for Windows "
                        "users) that you didn't specify the path to the "
                        "ImageMagick binary in file conf.py, or that the path "
                        "you specified is incorrect"))
            raise IOError(error)

        img = imread(tempfilename)
        img = cp.asarray(img)  # Convert to cupy array

        ImageClip.__init__(self, img, transparent=transparent)
        self.txt = txt
        self.color = color
        self.stroke_color = stroke_color

        if remove_temp:
            if os.path.exists(tempfilename):
                os.remove(tempfilename)
            if os.path.exists(temptxt):
                os.remove(temptxt)

    @staticmethod
    def list(arg):
        """Returns the list of all valid entries for the argument of
        ``TextClip`` given (can be ``font``, ``color``, etc...) """

        popen_params = {"stdout": sp.PIPE,
                        "stderr": DEVNULL,
                        "stdin": DEVNULL}

        if os.name == "nt":
            popen_params["creationflags"] = 0x08000000

        process = sp.Popen([get_setting("IMAGEMAGAGICK_BINARY"),
                            '-list', arg], **popen_params)
        result = process.communicate()[0]
        lines = result.splitlines()

        if arg == 'font':
            return [l.decode('UTF-8')[8:] for l in lines if l.startswith(b"  Font:")]
        elif arg == 'color':
            return [l.split(b" ")[0] for l in lines[2:]]
        else:
            raise Exception("Moviepy:Error! Argument must equal "
                            "'font' or 'color'")

    @staticmethod
    def search(string, arg):
        """Returns the of all valid entries which contain ``string`` for the
           argument ``arg`` of ``TextClip``, for instance

           >>> # Find all the available fonts which contain "Courier"
           >>> print ( TextClip.search('Courier', 'font') )

        """
        string = string.lower()
        names_list = TextClip.list(arg)
        return [name for name in names_list if string in name.lower()]
```

## 5、修改 moviepy/video/fx/fadein.py

```python
import cupy as cp

def fadein(clip, duration, initial_color=None):
    """
    Makes the clip progressively appear from some color (black by default),
    over ``duration`` seconds at the beginning of the clip. Can be used for
    masks too, where the initial color must be a number between 0 and 1.
    For cross-fading (progressive appearance or disappearance of a clip
    over another clip, see ``composition.crossfade``
    """

    if initial_color is None:
        initial_color = 0 if clip.ismask else [0, 0, 0]

    initial_color = cp.array(initial_color)

    def fl(gf, t):
        if t >= duration:
            return gf(t)
        else:
            fading = (1.0 * t / duration)
            return fading * gf(t) + (1 - fading) * initial_color

    return clip.fl(fl)
```

## 6、修改 moviepy/video/fx/fadeout.py

```python
import cupy as cp

from moviepy.decorators import requires_duration


@requires_duration
def fadeout(clip, duration, final_color=None):
    """
    Makes the clip progressively fade to some color (black by default),
    over ``duration`` seconds at the end of the clip. Can be used for
    masks too, where the final color must be a number between 0 and 1.
    For cross-fading (progressive appearance or disappearance of a clip
    over another clip, see ``composition.crossfade``
    """

    if final_color is None:
        final_color = 0 if clip.ismask else [0, 0, 0]

    final_color = cp.array(final_color)

    def fl(gf, t):
        if (clip.duration - t) >= duration:
            return gf(t)
        else:
            fading = 1.0 * (clip.duration - t) / duration
            return fading * gf(t) + (1 - fading) * final_color

    return clip.fl(fl)
```

## 7、修改 moviepy/video/io/ffmpeg_writer.py

```python
"""
On the long term this will implement several methods to make videos
out of VideoClips
"""

import os
import subprocess as sp

import cupy as cp
import numpy as np
from proglog import proglog

from moviepy.compat import DEVNULL, PY3
from moviepy.config import get_setting


class FFMPEG_VideoWriter:
    """ A class for FFMPEG-based video writing.

    A class to write videos using ffmpeg. ffmpeg will write in a large
    choice of formats.

    Parameters
    -----------

    filename
      Any filename like 'video.mp4' etc. but if you want to avoid
      complications it is recommended to use the generic extension
      '.avi' for all your videos.

    size
      Size (width,height) of the output video in pixels.

    fps
      Frames per second in the output video file.

    codec
      FFMPEG codec. It seems that in terms of quality the hierarchy is
      'rawvideo' = 'png' > 'mpeg4' > 'libx264'
      'png' manages the same lossless quality as 'rawvideo' but yields
      smaller files. Type ``ffmpeg -codecs`` in a terminal to get a list
      of accepted codecs.

      Note for default 'libx264': by default the pixel format yuv420p
      is used. If the video dimensions are not both even (e.g. 720x405)
      another pixel format is used, and this can cause problem in some
      video readers.

    audiofile
      Optional: The name of an audio file that will be incorporated
      to the video.

    preset
      Sets the time that FFMPEG will take to compress the video. The slower,
      the better the compression rate. Possibilities are: ultrafast,superfast,
      veryfast, faster, fast, medium (default), slow, slower, veryslow,
      placebo.

    bitrate
      Only relevant for codecs which accept a bitrate. "5000k" offers
      nice results in general.

    withmask
      Boolean. Set to ``True`` if there is a mask in the video to be
      encoded.

    """

    def __init__(self, filename, size, fps, codec="libx264", audiofile=None,
                 preset="medium", bitrate=None, withmask=False,
                 logfile=None, threads=None, ffmpeg_params=None):

        if logfile is None:
            logfile = sp.PIPE

        self.filename = filename
        self.codec = codec
        self.ext = self.filename.split(".")[-1]

        # order is important
        cmd = [
            get_setting("FFMPEG_BINARY"),
            '-hwaccel', 'cuvid',
            '-y',
            '-loglevel', 'error' if logfile == sp.PIPE else 'info',
            '-f', 'rawvideo',
            '-vcodec', 'rawvideo',
            '-s', '%dx%d' % (size[0], size[1]),
            '-pix_fmt', 'rgba' if withmask else 'rgb24',
            '-r', '%.02f' % fps,
            '-an', '-i', '-'
        ]
        if audiofile is not None:
            cmd.extend([
                '-i', audiofile,
                '-acodec', 'copy'
            ])
        cmd.extend([
            '-vcodec', codec,
            '-preset', preset,
        ])
        if ffmpeg_params is not None:
            cmd.extend(ffmpeg_params)
        if bitrate is not None:
            cmd.extend([
                '-b', bitrate
            ])

        if threads is not None:
            cmd.extend(["-threads", str(threads)])

        if ((codec == 'libx264') and
                (size[0] % 2 == 0) and
                (size[1] % 2 == 0)):
            cmd.extend([
                '-pix_fmt', 'yuv420p'
            ])
        cmd.extend([
            filename
        ])

        popen_params = {"stdout": DEVNULL,
                        "stderr": logfile,
                        "stdin": sp.PIPE}

        # This was added so that no extra unwanted window opens on windows
        # when the child process is created
        if os.name == "nt":
            popen_params["creationflags"] = 0x08000000  # CREATE_NO_WINDOW

        self.proc = sp.Popen(cmd, **popen_params)


    def write_frame(self, img_array):
        """ Writes one frame in the file."""
        try:
            if isinstance(img_array, cp.ndarray):
                img_array = cp.asnumpy(img_array)  # Convert to numpy array if necessary

            if PY3:
               self.proc.stdin.write(img_array.tobytes())
            else:
               self.proc.stdin.write(img_array.tostring())
        except IOError as err:
            _, ffmpeg_error = self.proc.communicate()
            error = (str(err) + ("\n\nMoviePy error: FFMPEG encountered "
                                 "the following error while writing file %s:"
                                 "\n\n %s" % (self.filename, str(ffmpeg_error))))

            if b"Unknown encoder" in ffmpeg_error:

                error = error+("\n\nThe video export "
                  "failed because FFMPEG didn't find the specified "
                  "codec for video encoding (%s). Please install "
                  "this codec or change the codec when calling "
                  "write_videofile. For instance:\n"
                  "  >>> clip.write_videofile('myvid.webm', codec='libvpx')")%(self.codec)

            elif b"incorrect codec parameters ?" in ffmpeg_error:

                 error = error+("\n\nThe video export "
                  "failed, possibly because the codec specified for "
                  "the video (%s) is not compatible with the given "
                  "extension (%s). Please specify a valid 'codec' "
                  "argument in write_videofile. This would be 'libx264' "
                  "or 'mpeg4' for mp4, 'libtheora' for ogv, 'libvpx for webm. "
                  "Another possible reason is that the audio codec was not "
                  "compatible with the video codec. For instance the video "
                  "extensions 'ogv' and 'webm' only allow 'libvorbis' (default) as a"
                  "video codec."
                  )%(self.codec, self.ext)

            elif  b"encoder setup failed" in ffmpeg_error:

                error = error+("\n\nThe video export "
                  "failed, possibly because the bitrate you specified "
                  "was too high or too low for the video codec.")

            elif b"Invalid encoder type" in ffmpeg_error:

                error = error + ("\n\nThe video export failed because the codec "
                  "or file extension you provided is not a video")


            raise IOError(error)

    def close(self):
        if self.proc:
            self.proc.stdin.close()
            if self.proc.stderr is not None:
                self.proc.stderr.close()
            self.proc.wait()

        self.proc = None

    # Support the Context Manager protocol, to ensure that resources are cleaned up.

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        self.close()

def ffmpeg_write_video(clip, filename, fps, codec="libx264", bitrate=None,
                       preset="medium", withmask=False, write_logfile=False,
                       audiofile=None, verbose=True, threads=None, ffmpeg_params=None,
                       logger='bar'):
    """ Write the clip to a videofile. See VideoClip.write_videofile for details
    on the parameters.
    """
    logger = proglog.default_bar_logger(logger)

    if write_logfile:
        logfile = open(filename + ".log", 'w+')
    else:
        logfile = None
    logger(message='Moviepy - Writing video %s\n' % filename)
    with FFMPEG_VideoWriter(filename, clip.size, fps, codec = codec,
                                preset=preset, bitrate=bitrate, logfile=logfile,
                                audiofile=audiofile, threads=threads,
                                ffmpeg_params=ffmpeg_params) as writer:

        nframes = int(clip.duration * fps)

        for t, frame in clip.iter_frames(logger=logger, with_times=True,
                                         fps=fps, dtype="uint8"):
            if withmask:
                mask = (255 * clip.mask.get_frame(t))
                if isinstance(mask, cp.ndarray):
                    mask = cp.asnumpy(mask)  # Convert to numpy array if necessary
                if mask.dtype != "uint8":
                    mask = mask.astype("uint8")
                frame = np.dstack([frame, mask])

            writer.write_frame(frame)

    if write_logfile:
        logfile.close()
    logger(message='Moviepy - Done !')


def ffmpeg_write_image(filename, image, logfile=False):
    """ Writes an image (HxWx3 or HxWx4 numpy array) to a file, using
        ffmpeg. """

    if isinstance(image, cp.ndarray):
        image = cp.asnumpy(image)  # Convert to numpy array if necessary

    if image.dtype != 'uint8':
          image = image.astype("uint8")

    cmd = [ get_setting("FFMPEG_BINARY"), '-y',
           '-s', "%dx%d" % (image.shape[:2][::-1]),
           "-f", 'rawvideo',
           '-pix_fmt', "rgba" if (image.shape[2] == 4) else "rgb24",
           '-i','-', filename]

    if logfile:
        log_file = open(filename + ".log", 'w+')
    else:
        log_file = sp.PIPE

    popen_params = {"stdout": DEVNULL,
                    "stderr": log_file,
                    "stdin": sp.PIPE}

    if os.name == "nt":
        popen_params["creationflags"] = 0x08000000

    proc = sp.Popen(cmd, **popen_params)
    out, err = proc.communicate(image.tobytes() if PY3 else image.tostring())

    if proc.returncode:
        err = "\n".join(["[MoviePy] Running : %s\n" % cmd,
                         "WARNING: this command returned an error:",
                         err.decode('utf8')])
        raise IOError(err)

    del proc
```

## 8、修改 moviepy/video/io/ffmpeg_reader.py

```python
"""
This module implements all the functions to read a video or a picture
using ffmpeg. It is quite ugly, as there are many pitfalls to avoid
"""

from __future__ import division

import logging
import os
import re
import subprocess as sp
import warnings

import cupy as cp
import numpy as np

from moviepy.compat import DEVNULL, PY3
from moviepy.config import get_setting  # ffmpeg, ffmpeg.exe, etc...
from moviepy.tools import cvsecs

logging.captureWarnings(True)


class FFMPEG_VideoReader:

    def __init__(self, filename, print_infos=False, bufsize=None,
                 pix_fmt="rgb24", check_duration=True,
                 target_resolution=None, resize_algo='bicubic',
                 fps_source='tbr'):

        self.filename = filename
        self.proc = None
        infos = ffmpeg_parse_infos(filename, print_infos, check_duration,
                                   fps_source)
        self.fps = infos['video_fps']
        self.size = infos['video_size']
        self.rotation = infos['video_rotation']

        if target_resolution:
            # revert the order, as ffmpeg used (width, height)
            target_resolution = target_resolution[1], target_resolution[0]

            if None in target_resolution:
                ratio = 1
                for idx, target in enumerate(target_resolution):
                    if target:
                        ratio = target / self.size[idx]
                self.size = (int(self.size[0] * ratio), int(self.size[1] * ratio))
            else:
                self.size = target_resolution
        self.resize_algo = resize_algo

        self.duration = infos['video_duration']
        self.ffmpeg_duration = infos['duration']
        self.nframes = infos['video_nframes']

        self.infos = infos

        self.pix_fmt = pix_fmt
        self.depth = 4 if pix_fmt == 'rgba' else 3

        if bufsize is None:
            w, h = self.size
            bufsize = self.depth * w * h + 100

        self.bufsize = bufsize
        self.initialize()

        self.pos = 1
        self.lastread = self.read_frame()

    def initialize(self, starttime=0):
        """Opens the file, creates the pipe. """

        self.close()  # if any

        if starttime != 0:
            offset = min(1, starttime)
            i_arg = ['-ss', "%.06f" % (starttime - offset),
                     '-i', self.filename,
                     '-ss', "%.06f" % offset]
        else:
            i_arg = ['-i', self.filename]

        cmd = ([get_setting("FFMPEG_BINARY")] + i_arg +
               ['-loglevel', 'error',
                '-f', 'image2pipe',
                '-vf', 'scale=%d:%d' % tuple(self.size),
                '-sws_flags', self.resize_algo,
                "-pix_fmt", self.pix_fmt,
                '-vcodec', 'rawvideo', '-'])
        popen_params = {"bufsize": self.bufsize,
                        "stdout": sp.PIPE,
                        "stderr": sp.PIPE,
                        "stdin": DEVNULL}

        if os.name == "nt":
            popen_params["creationflags"] = 0x08000000

        self.proc = sp.Popen(cmd, **popen_params)

    def skip_frames(self, n=1):
        """Reads and throws away n frames """
        w, h = self.size
        for i in range(n):
            self.proc.stdout.read(self.depth * w * h)
        self.pos += n

    def read_frame(self):
        w, h = self.size
        nbytes = self.depth * w * h

        s = self.proc.stdout.read(nbytes)
        if len(s) != nbytes:

            warnings.warn("Warning: in file %s, " % (self.filename) +
                          "%d bytes wanted but %d bytes read," % (nbytes, len(s)) +
                          "at frame %d/%d, at time %.02f/%.02f sec. " % (
                              self.pos, self.nframes,
                              1.0 * self.pos / self.fps,
                              self.duration) +
                          "Using the last valid frame instead.",
                          UserWarning)

            if not hasattr(self, 'lastread'):
                raise IOError(("MoviePy error: failed to read the first frame of "
                               "video file %s. That might mean that the file is "
                               "corrupted. That may also mean that you are using "
                               "a deprecated version of FFMPEG. On Ubuntu/Debian "
                               "for instance the version in the repos is deprecated. "
                               "Please update to a recent version from the website.") % (
                                  self.filename))

            result = self.lastread

        else:
            if hasattr(np, 'frombuffer'):
                result = cp.frombuffer(s, dtype='uint8')
            else:
                result = cp.fromstring(s, dtype='uint8')
            result.shape = (h, w, len(s) // (w * h))
            self.lastread = result

        return result

    def get_frame(self, t):
        """ Read a file video frame at time t.

        Note for coders: getting an arbitrary frame in the video with
        ffmpeg can be painfully slow if some decoding has to be done.
        This function tries to avoid fetching arbitrary frames
        whenever possible, by moving between adjacent frames.
        """

        pos = int(self.fps * t + 0.00001) + 1

        # Initialize proc if it is not open
        if not self.proc:
            self.initialize(t)
            self.pos = pos
            self.lastread = self.read_frame()

        if pos == self.pos:
            return self.lastread
        elif (pos < self.pos) or (pos > self.pos + 100):
            self.initialize(t)
            self.pos = pos
        else:
            self.skip_frames(pos - self.pos - 1)
        result = self.read_frame()
        self.pos = pos
        return result

    def close(self):
        if self.proc:
            self.proc.terminate()
            self.proc.stdout.close()
            self.proc.stderr.close()
            self.proc.wait()
            self.proc = None
        if hasattr(self, 'lastread'):
            del self.lastread

    def __del__(self):
        self.close()


def ffmpeg_read_image(filename, with_mask=True):
    """ Read an image file (PNG, BMP, JPEG...).

    Wraps FFMPEG_Videoreader to read just one image.
    Returns an ImageClip.

    This function is not meant to be used directly in MoviePy,
    use ImageClip instead to make clips out of image files.

    Parameters
    -----------

    filename
      Name of the image file. Can be of any format supported by ffmpeg.

    with_mask
      If the image has a transparency layer, ``with_mask=true`` will save
      this layer as the mask of the returned ImageClip

    """
    pix_fmt = 'rgba' if with_mask else "rgb24"
    reader = FFMPEG_VideoReader(filename, pix_fmt=pix_fmt, check_duration=False)
    im = reader.lastread
    del reader
    return im


def ffmpeg_parse_infos(filename, print_infos=False, check_duration=True,
                       fps_source='tbr'):
    """Get file infos using ffmpeg.

    Returns a dictionnary with the fields:
    "video_found", "video_fps", "duration", "video_nframes",
    "video_duration", "audio_found", "audio_fps"

    "video_duration" is slightly smaller than "duration" to avoid
    fetching the uncomplete frames at the end, which raises an error.

    """

    # open the file in a pipe, provoke an error, read output
    is_GIF = filename.endswith('.gif')
    cmd = [get_setting("FFMPEG_BINARY"), "-i", filename]
    if is_GIF:
        cmd += ["-f", "null", "/dev/null"]

    popen_params = {"bufsize": 10 ** 5,
                    "stdout": sp.PIPE,
                    "stderr": sp.PIPE,
                    "stdin": DEVNULL}

    if os.name == "nt":
        popen_params["creationflags"] = 0x08000000

    proc = sp.Popen(cmd, **popen_params)
    (output, error) = proc.communicate()
    infos = error.decode('utf8')

    del proc

    if print_infos:
        # print the whole info text returned by FFMPEG
        print(infos)

    lines = infos.splitlines()
    if "No such file or directory" in lines[-1]:
        raise IOError(("MoviePy error: the file %s could not be found!\n"
                       "Please check that you entered the correct "
                       "path.") % filename)

    result = dict()

    # get duration (in seconds)
    result['duration'] = None

    if check_duration:
        try:
            keyword = ('frame=' if is_GIF else 'Duration: ')
            # for large GIFS the "full" duration is presented as the last element in the list.
            index = -1 if is_GIF else 0
            line = [l for l in lines if keyword in l][index]
            match = re.findall("([0-9][0-9]:[0-9][0-9]:[0-9][0-9].[0-9][0-9])", line)[0]
            result['duration'] = cvsecs(match)
        except:
            raise IOError(("MoviePy error: failed to read the duration of file %s.\n"
                           "Here are the file infos returned by ffmpeg:\n\n%s") % (
                              filename, infos))

    # get the output line that speaks about video
    lines_video = [l for l in lines if ' Video: ' in l and re.search('\d+x\d+', l)]

    result['video_found'] = (lines_video != [])

    if result['video_found']:
        try:
            line = lines_video[0]

            # get the size, of the form 460x320 (w x h)
            match = re.search(" [0-9]*x[0-9]*(,| )", line)
            s = list(map(int, line[match.start():match.end() - 1].split('x')))
            result['video_size'] = s
        except:
            raise IOError(("MoviePy error: failed to read video dimensions in file %s.\n"
                           "Here are the file infos returned by ffmpeg:\n\n%s") % (
                              filename, infos))

        # Get the frame rate. Sometimes it's 'tbr', sometimes 'fps', sometimes
        # tbc, and sometimes tbc/2...
        # Current policy: Trust tbr first, then fps unless fps_source is
        # specified as 'fps' in which case try fps then tbr

        # If result is near from x*1000/1001 where x is 23,24,25,50,
        # replace by x*1000/1001 (very common case for the fps).

        def get_tbr():
            match = re.search("( [0-9]*.| )[0-9]* tbr", line)

            # Sometimes comes as e.g. 12k. We need to replace that with 12000.
            s_tbr = line[match.start():match.end()].split(' ')[1]
            if "k" in s_tbr:
                tbr = float(s_tbr.replace("k", "")) * 1000
            else:
                tbr = float(s_tbr)
            return tbr

        def get_fps():
            match = re.search("( [0-9]*.| )[0-9]* fps", line)
            fps = float(line[match.start():match.end()].split(' ')[1])
            return fps

        if fps_source == 'tbr':
            try:
                result['video_fps'] = get_tbr()
            except:
                result['video_fps'] = get_fps()

        elif fps_source == 'fps':
            try:
                result['video_fps'] = get_fps()
            except:
                result['video_fps'] = get_tbr()

        # It is known that a fps of 24 is often written as 24000/1001
        # but then ffmpeg nicely rounds it to 23.98, which we hate.
        coef = 1000.0 / 1001.0
        fps = result['video_fps']
        for x in [23, 24, 25, 30, 50]:
            if (fps != x) and abs(fps - x * coef) < .01:
                result['video_fps'] = x * coef

        if check_duration:
            result['video_nframes'] = int(result['duration'] * result['video_fps']) + 1
            result['video_duration'] = result['duration']
        else:
            result['video_nframes'] = 1
            result['video_duration'] = None

        # get the video rotation info.
        try:
            rotation_lines = [l for l in lines if 'rotate          :' in l and re.search('\d+$', l)]
            if len(rotation_lines):
                rotation_line = rotation_lines[0]
                match = re.search('\d+$', rotation_line)
                result['video_rotation'] = int(rotation_line[match.start(): match.end()])
            else:
                result['video_rotation'] = 0
        except:
            raise IOError(("MoviePy error: failed to read video rotation in file %s.\n"
                           "Here are the file infos returned by ffmpeg:\n\n%s") % (
                              filename, infos))

    lines_audio = [l for l in lines if ' Audio: ' in l]

    result['audio_found'] = lines_audio != []

    if result['audio_found']:
        line = lines_audio[0]
        try:
            match = re.search(" [0-9]* Hz", line)
            hz_string = line[match.start() + 1:match.end() - 3]  # Removes the 'hz' from the end
            result['audio_fps'] = int(hz_string)
        except:
            result['audio_fps'] = 'unknown'

    return result
```

## 9、修改 moviepy/video/io/VideoFileClip.py

```python
import os
import cupy as cp

from moviepy.audio.io.AudioFileClip import AudioFileClip
from moviepy.Clip import Clip
from moviepy.video.io.ffmpeg_reader import FFMPEG_VideoReader
from moviepy.video.VideoClip import VideoClip


class VideoFileClip(VideoClip):

    """

    A video clip originating from a movie file. For instance: ::

        >>> clip = VideoFileClip("myHolidays.mp4")
        >>> clip.close()
        >>> with VideoFileClip("myMaskVideo.avi") as clip2:
        >>>    pass  # Implicit close called by context manager.


    Parameters
    ------------

    filename:
      The name of the video file. It can have any extension supported
      by ffmpeg: .ogv, .mp4, .mpeg, .avi, .mov etc.

    has_mask:
      Set this to 'True' if there is a mask included in the videofile.
      Video files rarely contain masks, but some video codecs enable
      that. For istance if you have a MoviePy VideoClip with a mask you
      can save it to a videofile with a mask. (see also
      ``VideoClip.write_videofile`` for more details).

    audio:
      Set to `False` if the clip doesn't have any audio or if you do not
      wish to read the audio.

    target_resolution:
      Set to (desired_height, desired_width) to have ffmpeg resize the frames
      before returning them. This is much faster than streaming in high-res
      and then resizing. If either dimension is None, the frames are resized
      by keeping the existing aspect ratio.

    resize_algorithm:
      The algorithm used for resizing. Default: "bicubic", other popular
      options include "bilinear" and "fast_bilinear". For more information, see
      https://ffmpeg.org/ffmpeg-scaler.html

    fps_source:
      The fps value to collect from the metadata. Set by default to 'tbr', but
      can be set to 'fps', which may be helpful if importing slow-motion videos
      that get messed up otherwise.


    Attributes
    -----------

    filename:
      Name of the original video file.

    fps:
      Frames per second in the original file.


    Read docs for Clip() and VideoClip() for other, more generic, attributes.

    Lifetime
    --------

    Note that this creates subprocesses and locks files. If you construct one of these instances, you must call
    close() afterwards, or the subresources will not be cleaned up until the process ends.

    If copies are made, and close() is called on one, it may cause methods on the other copies to fail.

    """

    def __init__(self, filename, has_mask=False,
                 audio=True, audio_buffersize=200000,
                 target_resolution=None, resize_algorithm='bicubic',
                 audio_fps=44100, audio_nbytes=2, verbose=False,
                 fps_source='tbr'):

        VideoClip.__init__(self)

        # Make a reader
        pix_fmt = "rgba" if has_mask else "rgb24"
        self.reader = FFMPEG_VideoReader(filename, pix_fmt=pix_fmt,
                                         target_resolution=target_resolution,
                                         resize_algo=resize_algorithm,
                                         fps_source=fps_source)

        # Make some of the reader's attributes accessible from the clip
        self.duration = self.reader.duration
        self.end = self.reader.duration

        self.fps = self.reader.fps
        self.size = self.reader.size
        self.rotation = self.reader.rotation

        self.filename = self.reader.filename

        if has_mask:

            self.make_frame = lambda t: cp.asarray(self.reader.get_frame(t)[:,:,:3])
            mask_mf = lambda t: cp.asarray(self.reader.get_frame(t)[:,:,3]) / 255.0
            self.mask = (VideoClip(ismask=True, make_frame=mask_mf)
                         .set_duration(self.duration))
            self.mask.fps = self.fps

        else:

            self.make_frame = lambda t: cp.asarray(self.reader.get_frame(t))

        # Make a reader for the audio, if any.
        if audio and self.reader.infos['audio_found']:

            self.audio = AudioFileClip(filename,
                                       buffersize=audio_buffersize,
                                       fps=audio_fps,
                                       nbytes=audio_nbytes)

    def close(self):
        """ Close the internal reader. """
        if self.reader:
            self.reader.close()
            self.reader = None

        try:
            if self.audio:
                self.audio.close()
                self.audio = None
        except AttributeError:
            pass
```
