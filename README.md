# ubuntu-drivers-common

该软件包汇总和抽象了 Ubuntu 特定逻辑和有关第三方驱动程序包的知识，并为安装程序和驱动程序配置 GUI 提供 API。它还包含一些 NVIDIA 特定支持代码，以找到最合适的驱动程序版本（因为我们通常会装几个），以及设置专有 NVIDIA 和 FGLRX 软件包所使用的替代符号链接。

## 命令行接口

最简单的前端是 "ubuntu-drivers" 命令行工具。 您可以使用它来显示适用于当前系统的可用驱动程序包（`ubuntu-drivers list`），或者安装所有适合自动安装的驱动程序（`sudo ubuntu-drivers autoinstall`），这对于集成到安装程序中非常有用。

有关详细信息，请参阅 `ubuntu-drivers --help` 

## Python API

Python 模块 `UbuntuDrivers.detect` 提供了一些函数来检测系统的硬件、匹配的驱动程序包以及符合自动安装条件的包。

三个主要功能如下：

1. 哪些驱动程序包适用于此系统？

   `packages = UbuntuDrivers.detect.system_driver_packages()`

2. 哪些设备需要驱动程序，需要哪些软件包？

   `driver_info = UbuntuDrivers.detect.system_device_drivers()`

3. 哪一个驱动程序包适用于此硬件？

    ```python
    import apt
    apt_cache = apt.Cache
    apt_packages = UbuntuDrivers.detect.packages_for_modalias(apt_cache, modalias)
    ```

这些函数只使用 python-apt，不需要任何其他依赖、root权限、D-BUS调用等。

## 检测逻辑

将硬件映射到驱动程序包的主要方法是使用 modalias 模式。硬件设备导出一个 `"modalias"` sysfs 属性，例如：

```shell
$ cat /sys/devices/pci0000:00/0000:00:1b.0/modalias
pci:v00008086d00003B56sv000017AAsd0000215Ebc04sc03i00
```

内核模块用 modalias 模式（globs）声明它们可以处理哪些硬件，例如：

```shell
$ modinfo snd_hda_intel
[...]
alias:          pci:v00008086d*sv*sd*bc04sc03i00*
```

默认不安装的驱动程序包 (e. g. backports of drivers
from newer Linux packages, or the proprietary NVidia driver package
`nvidia-current`) have a `Modaliases:` package header which includes all
modalias patterns from all kernel modules that they ship. It is recommended to
add these headers to the package with `dh_modaliases(1)`.

`ubuntu-drivers-common` 使用这些包头将特定硬件（由 modalias标识）映射到覆盖该硬件的驱动程序包。

## 自定义检测插件

对于某些类型的驱动程序， modalias 检测方法不起作用。 例如 the "sl-modem-daemon" driver requires some checks in `/proc/asound/cards` and `aplay -l` to decide whether or not it applies to the system. These special cases can be put into a `detection plugin`, by adding a small piece of Python code to `/usr/share/ubuntu-drivers-common/detect/NAME.py` (shipped in `./detect-plugins/` in the `ubuntu-drivers-common` source). 他们需要导出一个方法

```python
   def detect(apt_cache):
      # do detection logic here
      return ['driver_package', ...]
```

它可以进行任何类型的检测，然后返回结果的适用于当前系统的包集。请注意，这不能依赖于拥有根权限。

## 自动打包测试

对于Ubuntu驱动程序的 `autopkg` 测试，在开发测试用例时可以使用以下命令：

```shell
$ PYTHONPATH=. tests/run test_ubuntu_drivers
```

建议总是在干净的环境中进行测试。使用 pbuilder chroot、sbuild chroot 或直接上传到 PPA，将减少由于特定系统而导致测试失败的可能性。
