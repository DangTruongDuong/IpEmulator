# Cấu trúc Dự án touchHLE (Phiên bản 0.2.3)

Tài liệu này mô tả chi tiết cấu trúc thư mục, chức năng của từng file/thư mục và kiến trúc tổng quan của dự án **touchHLE** — một trình giả lập cấp cao (High-Level Emulator - HLE) cho các ứng dụng iPhone OS (iOS) cổ điển, hỗ trợ chạy trên nhiều nền tảng bao gồm cả Android (APK).

---

## 🗺️ Sơ đồ Cấu trúc Thư mục Tổng quan

```text
touchHLE-0.2.3/
├── .cargo/                 # Cấu hình Cargo cho việc biên dịch chéo
├── .github/                # GitHub Actions CI/CD workflows
├── android/                # Mã nguồn dự án Android (APK wrapper)
│   ├── app/
│   │   ├── jni/            # Cấu hình JNI (Android.mk, Application.mk)
│   │   └── src/main/       # Manifest, Assets, Java/Kotlin code, Resources
│   └── gradle/             # Gradle wrapper
├── dev-docs/               # Tài liệu dành cho nhà phát triển (Build, Debug, Code style)
├── dev-scripts/            # Các script tự động hóa (Lint, Format, Package)
├── res/                    # Tài nguyên hình ảnh, icon ứng dụng
├── src/                    # MÃ NGUỒN CỐT LÕI (Rust + C++ wrapper)
│   ├── audio/              # Giả lập phần âm thanh và codec (openal_soft_wrapper, symphonia)
│   ├── cpu/                # Trình biên dịch động CPU (dynarmic_wrapper)
│   ├── dyld/               # Trình liên kết động giả lập (dyld, loader)
│   ├── environment/        # Quản lý luồng, đồng bộ hóa (mutex, thread scheduler)
│   ├── frameworks/         # Giả lập các thư viện hệ thống iOS (UIKit, Foundation, v.v.)
│   ├── fs/                 # Hệ thống tệp tin ảo cho iOS guest
│   ├── gles/               # Bộ dịch đồ họa OpenGL ES 1.1 -> OpenGL ES 2.0 / Modern GL
│   ├── image/              # Trình giải nén PVRTC và thư viện stb_image
│   ├── libc/               # Giả lập các hàm thư viện C tiêu chuẩn (stdio, math, pthread,...)
│   ├── mem/                # Quản lý bộ nhớ ảo của guest
│   ├── objc/               # Giả lập Runtime Objective-C (Message sending, classes, selectors)
│   ├── version/            # Crate quản lý phiên bản touchHLE
│   └── *.rs                # Các file logic hệ thống chính (lib.rs, bin.rs, abi.rs,...)
├── tests/                  # Kiểm thử tích hợp (Integration tests) và ứng dụng mẫu (TestApp)
├── touchHLE_apps/          # Thư mục mặc định chứa game (.ipa, .app)
├── touchHLE_dylibs/        # Thư mục chứa các thư viện động hệ thống iOS (.dylib gốc)
├── touchHLE_fonts/         # Thư mục chứa font chữ thay thế (Liberation, Noto Sans)
├── vendor/                 # Các thư viện bên ngoài làm submodule (SDL, Dynarmic, OpenAL Soft, stb)
├── build.rs                # Script build tự động sinh thông tin bản quyền và xử lý build NDK
├── Cargo.toml              # File cấu hình Cargo chính định nghĩa dependency
└── OPTIONS_HELP.txt        # Tài liệu hướng dẫn sử dụng các tham số dòng lệnh
```

---

## 📂 Chi tiết Từng Phần trong Dự án

### 1. Thư mục cốt lõi `src/` (Rust Code)

Đây là nơi chứa toàn bộ logic giả lập của touchHLE. Thay vì giả lập phần cứng của cả thiết bị hoặc chạy nhân hệ điều hành iOS gốc (như LLE - Low Level Emulation), touchHLE thực hiện **giả lập cấp cao (HLE)**: nó tải nhị phân Mach-O của game, tự quản lý bộ nhớ, thực hiện JIT dịch lệnh ARM sang x86_64/ARM64, và giả lập toàn bộ API hệ điều hành (libc, UIKit, Foundation) trực tiếp bằng Rust.

*   **`abi.rs`**: Định nghĩa Application Binary Interface (Giao diện nhị phân ứng dụng) để chuyển đổi kiểu dữ liệu, cấu trúc và con trỏ giữa bộ nhớ của ứng dụng 32-bit (Guest) và trình giả lập 64-bit (Host).
*   **`bin.rs`**: Điểm khởi đầu (entry point) của phiên bản chạy trên Máy tính (Windows, macOS, Linux).
*   **`lib.rs`**: Chứa hàm khởi chạy chính `main` và điểm khởi đầu `SDL_main` cho Android. Xử lý tham số dòng lệnh và nạp game.
*   **`app_picker.rs`**: Giao diện chọn game (GUI / Text) khi người dùng mở ứng dụng mà không truyền đường dẫn game cụ thể.
*   **`bundle.rs`**: Đọc định dạng file `.app` và `.ipa` của game iOS, xử lý metadata `Info.plist` và tài nguyên game.
*   **`cpu.rs` & `cpu/`**: Giả lập CPU.
    *   Sử dụng thư viện C++ **Dynarmic** (JIT compiler cho ARMv6/ARMv7-A 32-bit).
    *   `src/cpu/dynarmic_wrapper/`: Module C++ trung gian kết nối Rust với Dynarmic.
*   **`dyld.rs` & `dyld/`**: Trình liên kết động (Dynamic Linker) giả lập. Nạp các file `.dylib` từ thư mục `touchHLE_dylibs` vào bộ nhớ và liên kết chúng với mã nguồn ứng dụng.
*   **`environment.rs` & `environment/`**: Thiết lập môi trường chạy cho game. Quản lý trạng thái CPU, luồng (threads) và các cơ chế khóa đồng bộ (`mutex.rs`).
*   **`font.rs`**: Tải và kết xuất các font chữ có sẵn trong `touchHLE_fonts` để hiển thị chữ trong game.
*   **`fs.rs` & `fs/`**: Giả lập hệ thống tệp tin ảo của iOS (ví dụ: chuyển đường dẫn `/var/mobile/Documents` về thư mục lưu trữ thực tế trên thiết bị của người dùng).
*   **`gdb.rs`**: Tích hợp trình debug GDB để nhà phát triển có thể kết nối debug ứng dụng iOS đang giả lập.
*   **`matrix.rs`**: Hỗ trợ tính toán ma trận đồ họa 3D cho OpenGL.
*   **`mem.rs` & `mem/`**: Quản lý vùng nhớ ảo cấp cho ứng dụng 32-bit. Cấp phát vùng nhớ, đọc/ghi an toàn phòng tránh Null Pointer crash.
*   **`window.rs`**: Quản lý cửa sổ hiển thị, nhận các sự kiện chạm màn hình (touch events), chuột, bàn phím và điều khiển vẽ khung hình thông qua thư viện **SDL2**.

#### 📦 Thư mục con quan trọng trong `src/`:

*   **`src/frameworks/`**: Giả lập các Framework của Apple (Cocoa Touch). Đây là phần cốt lõi của phương pháp HLE.
    *   `uikit.rs`: Giả lập UIKit (Nút nhấn, view, màn hình, sự kiện chạm).
    *   `foundation.rs`: Giả lập Foundation (Chuỗi `NSString`, Mảng `NSArray`, Tự điển `NSDictionary`, Dữ liệu `NSData`, Luồng chạy `NSThread`, v.v.).
    *   `opengles.rs` & `gles/`: Bản dịch OpenGL ES 1.1 sang modern OpenGL. Game cổ điển sử dụng đồ họa 3D OpenGL ES 1.1, module này sẽ dịch các lời gọi hàm đó sang OpenGL ES 2.0+ hoặc Desktop OpenGL tương thích.
    *   `audio_toolbox.rs` & `core_audio_types.rs` & `openal.rs` & `audio/`: Xử lý âm thanh, giải mã định dạng IMA4/AAC/MP3 thông qua thư viện `symphonia` và phát âm thanh qua **OpenAL Soft**.
    *   `core_graphics.rs` & `core_animation.rs`: Vẽ hình học cơ bản và thực hiện hiệu ứng chuyển động.
    *   `game_kit.rs`, `store_kit.rs`, `media_player.rs`: Stub/giả lập trống cho các dịch vụ phụ của iOS.
*   **`src/libc/`**: Giả lập thư viện C chuẩn (System calls).
    *   Thay vì gọi trực tiếp libc của hệ thống host (vì địa chỉ bộ nhớ và kiến trúc khác biệt), touchHLE tự triển khai các hàm chuẩn như `malloc`, `free`, `pthread_create`, `fopen`, `socket`, `math` (sin, cos,...) và map chúng sang API của Rust.
*   **`src/objc/`**: Giả lập Objective-C Runtime.
    *   Do game iOS được viết bằng Objective-C, các lời gọi hàm đều qua `objc_msgSend`.
    *   Thư mục này mô phỏng các đối tượng class, selector, property, phương thức và cơ chế truyền thông điệp (messaging) của Objective-C ngay trong Rust.
*   **`src/image/`**: Xử lý nén/giải nén hình ảnh (PVRTC - định dạng nén texture của chip đồ họa PowerVR trên iPhone đời đầu) và thư viện stb_image.

---

### 2. Thư mục `android/` (Android APK Wrapper)

Thư mục này chứa một dự án Android Studio hoàn chỉnh để bao bọc nhân giả lập viết bằng Rust thành một file cài đặt APK Android.

*   **`android/app/build.gradle`**: Cấu hình biên dịch APK.
    *   Sử dụng plugin `cargo-ndk-android` để tự động kích hoạt Rust compiler biên dịch mã nguồn của touchHLE thành thư viện dùng chung `.so`.
    *   **Giới hạn kiến trúc**: Chỉ hỗ trợ cấu hình `arm64-v8a` (vì Dynarmic JIT yêu cầu CPU host là 64-bit ARM64/x86_64, do đó APK này không chạy được trên các thiết bị Android 32-bit cũ).
*   **`android/app/src/main/AndroidManifest.xml`**: Cấu hình quyền và thành phần của app Android.
*   **`android/app/src/main/java/org/touchhle/android/MainActivity.java`**:
    *   Kế thừa `SDLActivity` từ SDL2.
    *   Nạp thư viện `libSDL2.so` và `libtouchHLE.so`, sau đó kích hoạt hàm `SDL_main` trong Rust để chạy giả lập.
*   **`android/app/src/main/java/org/touchhle/android/DocumentsProvider.kt`**:
    *   Bộ cung cấp tài liệu tích hợp với hệ thống Android (DocumentsProvider API).
    *   Cho phép người dùng sử dụng ứng dụng "Files" của Android hoặc máy tính để sao chép game `.ipa` / `.app` vào thư mục lưu trữ ngoài của ứng dụng mà không bị chặn bởi các cơ chế bảo mật (scoped storage) của Android mới.

---

### 3. Thư mục `vendor/` (Các thành phần bên ngoài)

Các thư viện C/C++ cần thiết cho giả lập được nhúng trực tiếp hoặc dưới dạng Git Submodules:
*   **`dynarmic`**: CPU Recompiler cốt lõi.
*   **`SDL`**: Xử lý tạo cửa sổ, render màn hình và nhận input cho các nền tảng bao gồm cả Android.
*   **`openal-soft`**: Thư viện phát âm thanh 3D chất lượng cao.
*   **`PVRTDecompress`**: Giải mã định dạng nén ảnh PVRTC của iPhone.
*   **`stb`**: Các thư viện xử lý ảnh dạng header-only nhỏ gọn.

---

### 4. Các thư mục dữ liệu đi kèm (`touchHLE_*`)

*   **`touchHLE_apps/`**: Nơi người dùng đặt các ứng dụng iOS (file `.ipa` hoặc thư mục giải nén `.app`).
*   **`touchHLE_dylibs/`**: Chứa các file thư viện động hệ thống iOS nguyên bản được phân phối kèm (như `libgcc_s.1.dylib`, `libstdc++.6.0.9.dylib`, `libz.1.2.3.dylib`). Chúng được biên dịch cho ARMv6/v7 và sẽ được nạp trực tiếp bằng JIT.
*   **`touchHLE_fonts/`**: Chứa các font chữ mã nguồn mở Liberation và Noto Sans để giả lập các font hệ thống của iPhone (Arial, Helvetica, Courier, Times New Roman, v.v.).

---

## 🛠️ Hướng dẫn cho việc Chỉnh sửa & Phát triển sau này

Khi bạn muốn chỉnh sửa mã nguồn hoặc thêm tính năng mới, hãy tham khảo các tình huống sau:

### A. Thêm hoặc Sửa một Hàm Hệ thống iOS (HLE Frameworks)
Nếu game báo lỗi thiếu hàm UIKit hoặc Foundation (lỗi xuất hiện dạng *unimplemented function* trong log):
1.  Tìm file tương ứng trong [src/frameworks/](file:///f:/touchHLE-0.2.3/touchHLE-0.2.3/src/frameworks) (ví dụ: [uikit.rs](file:///f:/touchHLE-0.2.3/touchHLE-0.2.3/src/frameworks/uikit.rs) hoặc [foundation.rs](file:///f:/touchHLE-0.2.3/touchHLE-0.2.3/src/frameworks/foundation.rs)).
2.  Khai báo signature của hàm và triển khai logic bằng Rust. Sử dụng macro hệ thống để ánh xạ nó vào Objective-C runtime (xem các ví dụ khai báo Class/Method có sẵn trong các file này).

### B. Thay đổi Giao diện hoặc Sửa lỗi trên Android
Nếu bạn muốn cải thiện giao diện chọn file hoặc hành vi trên Android:
1.  Mở thư mục [android/](file:///f:/touchHLE-0.2.3/touchHLE-0.2.3/android) bằng **Android Studio**.
2.  Chỉnh sửa mã Kotlin trong [DocumentsProvider.kt](file:///f:/touchHLE-0.2.3/touchHLE-0.2.3/android/app/src/main/java/org/touchhle/android/DocumentsProvider.kt) hoặc Java trong [MainActivity.java](file:///f:/touchHLE-0.2.3/touchHLE-0.2.3/android/app/src/main/java/org/touchhle/android/MainActivity.java).
3.  Nếu thay đổi cách build hoặc cập nhật NDK, hãy cấu hình lại [build.gradle](file:///f:/touchHLE-0.2.3/touchHLE-0.2.3/android/app/build.gradle).

### C. Khắc phục lỗi Thư viện C chuẩn (libc)
Nếu game gọi một hàm POSIX/C chuẩn mà touchHLE chưa hỗ trợ:
1.  Xác định module tương ứng trong [src/libc/](file:///f:/touchHLE-0.2.3/touchHLE-0.2.3/src/libc) (như [posix_io.rs](file:///f:/touchHLE-0.2.3/touchHLE-0.2.3/src/libc/posix_io.rs) cho tệp tin, [pthread.rs](file:///f:/touchHLE-0.2.3/touchHLE-0.2.3/src/libc/pthread.rs) cho đa luồng, hoặc [stdio.rs](file:///f:/touchHLE-0.2.3/touchHLE-0.2.3/src/libc/stdio.rs)).
2.  Viết hàm thay thế bằng Rust để trả về dữ liệu tương thích với mong đợi của mã chạy trên iOS 32-bit.
