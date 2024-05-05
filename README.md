# kubernetes-cluster-setup

RHEL 系 OS を用いて kubernetes-cluster を作成しました。

- [公式手順](https://kubernetes.io/docs/setup/)

- [今回の構築手順](https://github.com/maeshinshin/kubernetes-cluster-setup/blob/main/setup/setup.md)

## 構成

mini pc 3 台

### cpu

```bash
# lscpu
Architecture:            x86_64
  CPU op-mode(s):        32-bit, 64-bit
  Address sizes:         39 bits physical, 48 bits virtual
  Byte Order:            Little Endian
CPU(s):                  4
  On-line CPU(s) list:   0-3
Vendor ID:               GenuineIntel
  Model name:            Intel(R) N100
    CPU family:          6
    Model:               190
    Thread(s) per core:  1
    Core(s) per socket:  4
    Socket(s):           1
    Stepping:            0
    CPU(s) scaling MHz:  97%
    CPU max MHz:         3400.0000
    CPU min MHz:         700.0000
    BogoMIPS:            1612.80
```

### memory

```bash
# free -mh
               total        used        free      shared  buff/cache   available
Mem:            15Gi       1.1Gi       8.9Gi        13Mi       5.8Gi        14Gi
Swap:             0B          0B          0B
```

### os

```bash
# cat /etc/os-release
NAME="Fedora Linux"
VERSION="39 (Server Edition)"
ID=fedora
VERSION_ID=39
VERSION_CODENAME=""
PLATFORM_ID="platform:f39"
PRETTY_NAME="Fedora Linux 39 (Server Edition)"
ANSI_COLOR="0;38;2;60;110;180"
LOGO=fedora-logo-icon
CPE_NAME="cpe:/o:fedoraproject:fedora:39"
HOME_URL="https://fedoraproject.org/"
DOCUMENTATION_URL="https://docs.fedoraproject.org/en-US/fedora/f39/system-administrators-guide/"
SUPPORT_URL="https://ask.fedoraproject.org/"
BUG_REPORT_URL="https://bugzilla.redhat.com/"
REDHAT_BUGZILLA_PRODUCT="Fedora"
REDHAT_BUGZILLA_PRODUCT_VERSION=39
REDHAT_SUPPORT_PRODUCT="Fedora"
REDHAT_SUPPORT_PRODUCT_VERSION=39
SUPPORT_END=2024-11-12
VARIANT="Server Edition"
VARIANT_ID=server
```
