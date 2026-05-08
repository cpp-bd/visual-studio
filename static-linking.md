**Visual Studio-এ Static Linking পুরোপুরি A থেকে Z বুঝিয়ে দিলাম (২০২৬ ভার্সন অনুযায়ী)**

Static Linking মানে হলো তোমার প্রজেক্টের **EXE ফাইলের মধ্যে** লাইব্রেরির কোড পুরোপুরি ঢুকিয়ে দেওয়া। ফলে:

- কোনো `.dll` ফাইল দরকার হয় না।
- অন্য কম্পিউটারে শুধু EXE কপি করলেই চলে।
- সাইজ একটু বড় হয়, কিন্তু পোর্টেবল এবং ডিপেন্ডেন্সি-ফ্রি।

২০২৬ সালেও **vcpkg** হলো সবচেয়ে সহজ ও শক্তিশালী উপায়।

### ১. vcpkg ইনস্টল করা (যদি না থাকে)

```bat
git clone https://github.com/microsoft/vcpkg.git C:\vcpkg
cd C:\vcpkg
bootstrap-vcpkg.bat
```

(পছন্দমতো পাথে রাখতে পারো, যেমন `D:\tools\vcpkg`)

### ২. Visual Studio-এর সাথে ইন্টিগ্রেট করা

```bat
vcpkg integrate install
```

এটি করলে Visual Studio স্বয়ংক্রিয়ভাবে vcpkg ডিটেক্ট করবে।

### ৩. Static লাইব্রেরি ইনস্টল করা (সবচেয়ে গুরুত্বপূর্ণ ধাপ)

**Dynamic (ভুল):**
```bat
vcpkg install libwebp:x64-windows
```

**Static (সঠিক):**
```bat
vcpkg install libwebp:x64-windows-static
```

অন্যান্য লাইব্রেরির জন্যও একই:
- `openssl:x64-windows-static`
- `opencv4:x64-windows-static`
- `ffmpeg:x64-windows-static` ইত্যাদি।

**টিপ:** প্রথমবার অনেক সময় লাগতে পারে কারণ সোর্স থেকে বিল্ড হয়।

### ৪. ইনস্টল হয়েছে কিনা চেক করো

```bat
dir C:\vcpkg\installed\x64-windows-static\lib
dir C:\vcpkg\installed\x64-windows-static\include
```

`.lib` ফাইলগুলো দেখা উচিত।

### ৫. নতুন প্রজেক্ট তৈরি করো

- Visual Studio 2026 ওপেন করো।
- **Console App** বা **Empty Project** (C++) তৈরি করো।
- **Solution Platform** → **x64** সেট করো (Win32 নয়!)।

### ৬. প্রজেক্ট প্রপার্টিজ সেটিংস (সবচেয়ে গুরুত্বপূর্ণ)

**Project → Properties** (Alt + Enter):

#### A. Include ডিরেক্টরি
**C/C++ → General → Additional Include Directories**
```
C:\vcpkg\installed\x64-windows-static\include
```

#### B. Library ডিরেক্টরি
**Linker → General → Additional Library Directories**
```
C:\vcpkg\installed\x64-windows-static\lib
```

#### C. Dependencies (Additional Dependencies)
**Linker → Input → Additional Dependencies**

যেমন libwebp-এর জন্য:
```
libwebp.lib
libsharpyuv.lib
```
(দরকারে আরও: `libwebpdecoder.lib`, `libwebpdemux.lib` ইত্যাদি)

#### D. Static Runtime Library (খুবই ইম্পর্ট্যান্ট!)

**C/C++ → Code Generation → Runtime Library**

- **Debug** → **Multi-threaded Debug (/MTd)**
- **Release** → **Multi-threaded (/MT)**

এটি না করলে VC++ Runtime DLL ডিপেন্ডেন্সি থেকে যাবে।

### ৭. টেস্ট কোড

```cpp
#include <webp/encode.h>
#include <iostream>

int main() {
    std::cout << "WebP Static Linking Successful!\n";
    std::cout << "Version: " << WebPGetDecoderVersion() << std::endl;
    return 0;
}
```

### ৮. Build & Run

**Build → Rebuild Solution**  
EXE পাবে: `x64\Release\YourApp.exe`

এই EXE অন্য যেকোনো Windows PC-তে (একই আর্কিটেকচার) চলবে — কোনো DLL ছাড়াই।

---

### সাধারণ সমস্যা ও সমাধান (Troubleshooting)

| সমস্যা | সমাধান |
|-------|--------|
| LNK2019 / unresolved external | লাইব্রেরি নাম ঠিক আছে কিনা চেক করো + x64 প্ল্যাটফর্ম |
| DLL dependency থেকে যায় | Runtime Library `/MT` করো |
| Header ফাইল পায় না | Include path ভুল |
| অনেকগুলো .lib দরকার | প্রজেক্টের `.vcxproj` ফাইলে `<AdditionalDependencies>` দেখো |
| VS 2026-এ vcpkg ইন্টিগ্রেশন ইস্যু | `vcpkg integrate install` আবার চালাও |

### ভবিষ্যতে নতুন লাইব্রেরির জন্য ইউনিভার্সাল ফর্মুলা

1. `vcpkg install package_name:x64-windows-static`
2. Include path যোগ করো
3. Lib path যোগ করো
4. প্রয়োজনীয় `.lib` ফাইলগুলো Additional Dependencies-এ যোগ করো
5. Runtime Library `/MT` (Release) / `/MTd` (Debug)
6. Build

**বোনাস টিপস (২০২৬):**
- **Manifest mode** ব্যবহার করলে (`vcpkg.json`) আরও আধুনিক ও সুবিধাজনক।
- CMake প্রজেক্ট হলে `vcpkg.cmake` toolchain ফাইল ব্যবহার করো — ম্যানুয়াল পাথ সেট করতে হয় না।
- একাধিক triplet একসাথে রাখতে চাইলে overlay triplets ব্যবহার করো।

চাইলে কোনো নির্দিষ্ট লাইব্রেরি (যেমন OpenCV, FFmpeg, SDL2) নিয়ে বিস্তারিত স্টেপ দিতে পারি। বলো! 

এভাবে ফলো করলে তোমার অ্যাপ্লিকেশন পুরোপুরি **statically linked** হয়ে যাবে। শুভ কোডিং! 🚀
