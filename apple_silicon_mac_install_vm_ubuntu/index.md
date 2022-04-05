# Apple Silicon M1的Mac电脑安装ubuntu虚拟机



为了学习linux内核，安装一个ubuntu虚拟机比较方便。

## 安装虚拟机软件
Parallel Desktop收费，pass。

[VMWare Fusion](https://customerconnect.vmware.com/downloads/get-download?downloadGroup=FUS-PUBTP-2021H1)还是技术预览版。

最终选择了[UTM](https://mac.getutm.app/)。

## 下载安装ubuntu
[https://mac.getutm.app/gallery/ubuntu-20-04](https://mac.getutm.app/gallery/ubuntu-20-04)

按照上面的指引安装即可。

## 修改分辨率
目前在用的MacBook Pro 14的分辨率为3024x1964，该分辨率还包含了刘海那一横条部分。ubuntu系统使用16:10的分辨率就好，刘海那一条置黑，因此需要的分辨率为3024x1890。

首次设置：

```bash
cvt 3024 1890
sudo xrandr --newmode "3024x1890_60.00"  488.50  3024 3264 3592 4160  1890 1893 1899 1958 -hsync +vsync
sudo xrandr --addmode Virtual-1 "3024x1890_60.00"
sudo xrandr --output Virtual-1 --mode "3024x1890_60.00"
```
为了避免重启失效，把以下代码加入`~/.profile`中：

```bash
cvt 3024 1890
xrandr --newmode "3024x1890_60.00"  488.50  3024 3264 3592 4160  1890 1893 1899 1958 -hsync +vsync
xrandr --addmode Virtual-1 "3024x1890_60.00"
xrandr --output Virtual-1 --mode "3024x1890_60.00"
```


