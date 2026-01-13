 ---
  全体像

  このコードは、PCを起動して画面に斜めの白い線を描くだけのシンプルなOSカーネルです。

  ---

  1. ヘッダーファイルのインクルード（3〜6行目）

  #include <limine.h>      // Limineブートローダーとの通信用
  #include <stdbool.h>     // true/false を使うため
  #include <stddef.h>      // size_t, NULL を使うため
  #include <stdint.h>      // uint8_t, uint32_t などの型を使うため

  ---

  1. Limineへのリクエスト（8〜21行目）

  __attribute__((used, section(".limine_requests")))
  static volatile struct limine_framebuffer_request framebuffer_request = {
      .id = LIMINE_FRAMEBUFFER_REQUEST,
      .revision = 0
  };

  何をしているか：

- ブートローダー（Limine）に「画面（フレームバッファ）を使いたい」と伝えています
- __attribute__((used, section(...))) は、このデータを特別なメモリ領域に配置する指示です
- Limineがこのリクエストを見つけて、フレームバッファの情報を返してくれます

  ---

  1. メモリ操作関数（23〜72行目）

  OSには標準ライブラリがないため、基本的な関数を自分で実装しています。

  memcpy - メモリのコピー

  void *memcpy(void*dest, const void *src, size_t n) {
      for (size_t i = 0; i < n; i++) {
          pdest[i] = psrc[i];  // 1バイトずつコピー
      }
  }

  memset - メモリを特定の値で埋める

  void *memset(void*s, int c, size_t n) {
      for (size_t i = 0; i < n; i++) {
          p[i] = (uint8_t)c;  // 全てのバイトを c で埋める
      }
  }

  memmove - 重複領域対応のコピー

  void *memmove(void*dest, const void *src, size_t n) {
      if (src > dest) {
          // 前から後ろへコピー
      } else if (src < dest) {
          // 後ろから前へコピー（重複対策）
      }
  }
  memcpyと違い、コピー元とコピー先が重なっていても正しく動きます。

  memcmp - メモリの比較

  int memcmp(const void *s1, const void*s2, size_t n) {
      // 同じなら0、違えば-1か1を返す
  }

  ---

  1. hcf関数（74〜78行目）

  static void hcf(void) {
      for (;;) {
          asm("hlt");  // CPUを停止させる
      }
  }

  hcf = "Halt and Catch Fire"（停止して燃え尽きる）

  無限ループでCPUを休止状態にします。OSが終了したとき、エラーが発生したときに呼ばれます。

  ---

  1. kmain関数（80〜98行目）- カーネルのメイン

  void kmain(void) {
  これがOSのエントリポイント（最初に実行される関数）です。

  エラーチェック

  if (!LIMINE_BASE_REVISION_SUPPORTED) {
      hcf();  // Limineのバージョンが合わなければ停止
  }
  if (framebuffer_request.response == NULL ||
      framebuffer_request.response->framebuffer_count < 1) {
      hcf();  // フレームバッファが取得できなければ停止
  }

  フレームバッファの取得

  struct limine_framebuffer *framebuffer =
      framebuffer_request.response->framebuffers[0];
  Limineが用意してくれた画面情報を取得します。

  画面に線を描く

  for (size_t i = 0; i < 100; i++) {
      volatile uint32_t *fb_ptr = framebuffer->address;
      fb_ptr[i* (framebuffer->pitch / 4) + i] = 0xffffff;
  }

  解説：

- framebuffer->address = 画面メモリの先頭アドレス
- framebuffer->pitch = 1行あたりのバイト数
- 0xffffff = 白色（RGB各8ビット、全部最大値）
- i * (pitch/4) + i = 斜め下に進む座標計算

  (0,0)に白
        (1,1)に白
              (2,2)に白
                    ... と100ピクセル分

  これで左上から右下への斜めの白い線が描かれます。

  ---

  1. リンカスクリプト（linker.lds）

  ENTRY(kmain)           // 開始地点はkmain関数
  . = 0xffffffff80000000; // カーネルを高位アドレスに配置

  カーネルをメモリのどこに配置するかを決めています。

  ---
  まとめ

  起動の流れ：

  1. PC電源ON → BIOS/UEFI
  2. Limineブートローダーが起動
  3. Limineがカーネル（このコード）を読み込む
  4. Limineがフレームバッファを準備
  5. kmain()が呼ばれる
  6. 画面に斜めの白線を描画
  7. CPUを停止（hcf()）

  これが自作OSの最初の一歩です。ここから文字表示、キーボード入力、メモリ管理などを追加していくことで本格的なOSになっていきます。1

### ビルド方法

qemu-system-x86_64 -cdrom image.iso -boot d -m 512M -cpu qemu64 -enable-kvm
