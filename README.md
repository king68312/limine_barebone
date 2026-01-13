# Limine Barebones Kernel

シンプルな自作OSカーネルのサンプルです。PCを起動して画面に斜めの白い線を描きます。

## 概要

このコードは、Limineブートローダーを使用して起動し、フレームバッファに直接描画を行う最小限のカーネルです。

## 起動の流れ

```
PC電源ON → BIOS/UEFI → Limineブートローダー → カーネル(kmain) → 画面描画 → CPU停止
```

## コード解説

### 1. ヘッダーファイルのインクルード

```c
#include <limine.h>      // Limineブートローダーとの通信用
#include <stdbool.h>     // true/false を使うため
#include <stddef.h>      // size_t, NULL を使うため
#include <stdint.h>      // uint8_t, uint32_t などの型を使うため
```

### 2. Limineへのリクエスト

```c
__attribute__((used, section(".limine_requests")))
static volatile struct limine_framebuffer_request framebuffer_request = {
    .id = LIMINE_FRAMEBUFFER_REQUEST,
    .revision = 0
};
```

- ブートローダー（Limine）に「画面（フレームバッファ）を使いたい」と伝えています
- `__attribute__((used, section(...)))` は、このデータを特別なメモリ領域に配置する指示です
- Limineがこのリクエストを見つけて、フレームバッファの情報を返してくれます

### 3. メモリ操作関数

OSには標準ライブラリがないため、基本的な関数を自分で実装しています。

#### memcpy - メモリのコピー

```c
void *memcpy(void *dest, const void *src, size_t n) {
    for (size_t i = 0; i < n; i++) {
        pdest[i] = psrc[i];  // 1バイトずつコピー
    }
}
```

#### memset - メモリを特定の値で埋める

```c
void *memset(void *s, int c, size_t n) {
    for (size_t i = 0; i < n; i++) {
        p[i] = (uint8_t)c;  // 全てのバイトを c で埋める
    }
}
```

#### memmove - 重複領域対応のコピー

```c
void *memmove(void *dest, const void *src, size_t n) {
    if (src > dest) {
        // 前から後ろへコピー
    } else if (src < dest) {
        // 後ろから前へコピー（重複対策）
    }
}
```

`memcpy`と違い、コピー元とコピー先が重なっていても正しく動きます。

#### memcmp - メモリの比較

```c
int memcmp(const void *s1, const void *s2, size_t n) {
    // 同じなら0、違えば-1か1を返す
}
```

### 4. hcf関数

```c
static void hcf(void) {
    for (;;) {
        asm("hlt");  // CPUを停止させる
    }
}
```

**hcf** = "Halt and Catch Fire"（停止して燃え尽きる）

無限ループでCPUを休止状態にします。OSが終了したとき、エラーが発生したときに呼ばれます。

### 5. kmain関数 - カーネルのメイン

```c
void kmain(void) {
    // これがOSのエントリポイント（最初に実行される関数）です
}
```

#### エラーチェック

```c
if (!LIMINE_BASE_REVISION_SUPPORTED) {
    hcf();  // Limineのバージョンが合わなければ停止
}
if (framebuffer_request.response == NULL ||
    framebuffer_request.response->framebuffer_count < 1) {
    hcf();  // フレームバッファが取得できなければ停止
}
```

#### フレームバッファの取得

```c
struct limine_framebuffer *framebuffer =
    framebuffer_request.response->framebuffers[0];
```

Limineが用意してくれた画面情報を取得します。

#### 画面に線を描く

```c
for (size_t i = 0; i < 100; i++) {
    volatile uint32_t *fb_ptr = framebuffer->address;
    fb_ptr[i * (framebuffer->pitch / 4) + i] = 0xffffff;
}
```

| 項目 | 説明 |
|------|------|
| `framebuffer->address` | 画面メモリの先頭アドレス |
| `framebuffer->pitch` | 1行あたりのバイト数 |
| `0xffffff` | 白色（RGB各8ビット、全部最大値） |
| `i * (pitch/4) + i` | 斜め下に進む座標計算 |

```
(0,0)に白
      (1,1)に白
            (2,2)に白
                  ... と100ピクセル分
```

これで左上から右下への斜めの白い線が描かれます。

### 6. リンカスクリプト（linker.lds）

```ld
ENTRY(kmain)            // 開始地点はkmain関数
. = 0xffffffff80000000; // カーネルを高位アドレスに配置
```

カーネルをメモリのどこに配置するかを決めています。

## ビルド方法

```bash
# ISOイメージをQEMUで起動
qemu-system-x86_64 -cdrom image.iso -boot d -m 512M -cpu qemu64 -enable-kvm
```

## 次のステップ

これが自作OSの最初の一歩です。ここから以下の機能を追加していくことで本格的なOSになっていきます：

- 文字表示
- キーボード入力
- メモリ管理
- プロセス管理
- ファイルシステム

## 参考リンク

- [Limine Bootloader](https://github.com/limine-bootloader/limine)
- [OSDev Wiki](https://wiki.osdev.org/)
