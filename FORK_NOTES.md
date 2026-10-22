# 本地分支说明（zhyiheihei/crosspoint-onepage）

本仓是 `MoveCall/crosspoint-onepage`（OnePage / ESP32-C61 电纸书，CrossPoint Reader 的 C61 移植）
的个人维护分支。上游提交完整保留，本地只在 HEAD 之上叠加必要修复，便于随时
`git rebase upstream/main` 跟进。

## 与上游的唯一实质差异：mbedTLS 走 PSRAM 分配

**症状**：OPDS/OTA 等任何 HTTPS 请求失败，串口报

```
E esp-tls-mbedtls: mbedtls_ssl_setup returned -0x7F00
E esp-tls: create_ssl_handle failed
[ERR] [HTTP] open failed: ESP_ERR_HTTP_CONNECT
```

`-0x7F00` 是 `MBEDTLS_ERR_SSL_ALLOC_FAILED`。

**根因**（实测，非推测）：这台 C61 有 8 MB PSRAM 且已正常初始化
（`esp_psram_is_initialized()=1`、`esp_psram_get_size()=8388608`），但 Wi-Fi 起来后
片内 SRAM 只剩约 9.2 KB 最大连续块，而 TLS 握手需要约 32 KB。PSRAM 够大却用不上，因为
IDF 默认 `CONFIG_MBEDTLS_INTERNAL_MEM_ALLOC=y`，mbedTLS 只从片内 RAM 取内存。

**为什么改 sdkconfig 没用**：arduino 的 IDF 组件库是预编译产物，不随 `custom_sdkconfig`
重编。反汇编 `libmbedcrypto.a` 的 `esp_mbedtls_mem_calloc` 可以看到它硬编码
`MALLOC_CAP_INTERNAL | MALLOC_CAP_8BIT`（0x804），而外部分配应为 `MALLOC_CAP_SPIRAM`
（0x404）。症状是改完配置重编两次，`firmware.bin` 字节数完全相同。

**修法**：在 `setup()` 最前面、任何 TLS 使用之前，用 mbedTLS 官方接口替换分配器：

```cpp
static void* psramCalloc(size_t n, size_t size) {
  void* p = heap_caps_calloc(n, size, MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT);
  if (p == nullptr) p = heap_caps_calloc(n, size, MALLOC_CAP_DEFAULT);
  return p;
}
...
mbedtls_platform_set_calloc_free(psramCalloc, heap_caps_free);
```

`platformio.ini` 里同时保留了 `CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC=y`：当前对预编译库无效，
但一旦哪天库改为从源码构建就会生效，留着与运行期修改一致。

另外 `setup()` 里有两行诊断日志（片内/PSRAM 总量与堆占用），排查完可以删。

**验证**：`https://books.zhyi.xin/feed` 可浏览、分类可翻页、EPUB 可下载：

```
[OPDS] Fetching: https://books.zhyi.xin/feed
[OPDS] Fetching: https://books.zhyi.xin/index.php/feed/languages/1/Chinese
[HTTP] Downloaded 931374 bytes
[BookCache] Done checking metadata cache for: /Guanzhong Luo - 三國志演義.epub
```

## 构建与刷机

本机（NixOS）直接跑 PlatformIO 会撞动态链接器与 `/tmp` 沙箱问题，用容器构建：

```bash
podman run --rm --network=host -v "$PWD":/work -v "$HOME/.cache/pio-c61":/root/.platformio -w /work \
  docker.io/library/ubuntu:24.04 bash -c '
    apt-get -o Acquire::http::Proxy="DIRECT" update -qq &&
    apt-get install -y -qq python3 python3-venv git ca-certificates &&
    python3 -m venv /opt/pio &&
    /opt/pio/bin/pip install -q -U "https://github.com/pioarduino/platformio-core/archive/refs/tags/v6.1.19.zip" &&
    /opt/pio/bin/pio run -e onepage'
```

刷机（先 `flash-id` 看 Flash 厂家：`85` = Puya 必须 `--no-stub`）：

```bash
esptool --chip esp32c61 -p /dev/ttyACM0 -b 921600 write-flash \
  --flash-mode dio --flash-freq 80m --flash-size 16MB 0x10000 .pio/build/onepage/firmware.bin
```

设备休眠会掉 USB（`/dev/ttyACM0` 消失），刷机/抓日志前须保持亮屏。

## 硬件实测参数

| 项 | 值 |
| --- | --- |
| 芯片 | ESP32-C61（ROM `esp32c61-eco4-20250829`） |
| Flash | 16 MB，厂家字节 `68` |
| PSRAM | 8 MB |
| 片内 SRAM 堆 | 159,558 字节 |
| USB | `303a:1001` USB-Serial/JTAG，115200 波特 |

注意 `ESP.getHeapSize()` 只统计片内 RAM（打印 159558 容易误判成"PSRAM 没起来"），
判断 PSRAM 要用 `heap_caps_get_total_size(MALLOC_CAP_SPIRAM)` 或 `esp_psram_get_size()`。
