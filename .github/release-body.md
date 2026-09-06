## ZC-328 Armbian 镜像

- **Armbian tag**: v26.11.0-trunk.30
- **Board**: ZC-328 (Rockchip RK3288)
- **Branch**: current
- **压缩**: xz（IMAGE_XZ_COMPRESSION_RATIO=9）
- **系统变体**: noble / resolute / bookworm / trixie × minimal / cli / xfce 桌面
- **deb 软件包**: 板级内核 / DTB / headers 包已随本 release 附带

## 默认登录账号

Armbian 镜像默认凭据：

- **用户名**: root
- **密码**: 1234

> 首次登录会强制要求设置新密码。建议同时创建普通用户并 apt-mark hold 板级内核/U-Boot 包，避免升级被普通源覆盖（详见仓库 README）。
