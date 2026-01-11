### 以下修改在moviepy 1.0.3，CUDA为12.1.1，Pytorch为Windows Cuda12.1的环境下测试通过。确实比CPU生成有提升，大约40%。

moviepy默认的安装路径为：
----------------

```
C:\Users\Administrator\AppData\Local\Programs\Python\Python312\Lib\site-packages\moviepy
```

1、修改源文件moviepy/video/tools/drawing.py
-------------------------------------

修改blit为blit\_gpu

```
import numpy as np
import torch

def blit_gpu(im1, im2, pos=None, mask=None, ismask=False):
    device = torch.device("cuda:0" if torch.cuda.is_available() else "cpu")

    if pos is None:
        pos = [0, 0]

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

    if not isinstance(im1, torch.Tensor):  # 将im1转换为torch.Tensor
        im1 = torch.tensor(im1, device=device)
    if not isinstance(im2, torch.Tensor):  # 将im2转换为torch.Tensor
        im2 = torch.tensor(im2, device=device)

    # 提取需要blit的区域
    blitted = im1[y1:y2, x1:x2].to(device)

    new_im2 = im2.clone().to(device)

    if mask is None:
        new_im2[yp1:yp2, xp1:xp2] = blitted
    else:
        if not isinstance(mask, torch.Tensor):  # 将mask转换为torch.Tensor
            mask = torch.tensor(mask[y1:y2, x1:x2], device=device)
        else:
            mask = mask[y1:y2, x1:x2].to(device)

        if len(im1.shape) == 3:
            mask = mask.unsqueeze(-1).repeat(1, 1, 3)

        blit_region = new_im2[yp1:yp2, xp1:xp2]
        new_im2[yp1:yp2, xp1:xp2] = mask * blitted + (1 - mask) * blit_region

    return new_im2.cpu().numpy().astype("uint8") if not ismask else new_im2.cpu().numpy()
```

2、修改源文件moviepy/video/VideoClip.py
---------------------------------

在文件中头部修改

```
from .tools.drawing import blit_gpu
```

修改第565行返回为下面的代码

```
return blit_gpu(img, picture, pos, mask=mask, ismask=self.ismask)
```

3、修改源文件moviepy\Clip.py
----------------------

把iter\_frames方法的以下部分

```
if (dtype is not None) and (frame.dtype != dtype):
    frame = frame.astype(dtype)
```

修改为：

```
if (dtype is not None) and (frame.dtype != dtype):
    frame = frame.cpu().numpy().astype(dtype)
```