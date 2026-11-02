# Kaleidoscope: Biên dịch sang mã đối tượng

## Giới thiệu chương 8

Chào mừng bạn đến với chương 8 của loạt hướng dẫn "[Hiện thực một ngôn ngữ với LLVM](index.md)". Chương này mô tả cách biên dịch ngôn ngữ của chúng ta xuống thành các tệp đối tượng (object file).

## Chọn kiến trúc đích

LLVM hỗ trợ sẵn biên dịch chéo (cross-compilation). Bạn có thể biên dịch cho kiến trúc của máy hiện tại, hoặc biên dịch cho các kiến trúc khác một cách dễ dàng không kém. Trong hướng dẫn này, chúng ta sẽ nhắm đến máy hiện tại.

Để chỉ định kiến trúc mà bạn muốn nhắm đến, chúng ta dùng một chuỗi gọi là "target triple" (bộ ba đích). Chuỗi này có dạng `<arch><sub>-<vendor>-<sys>-<abi>` (xem [tài liệu về biên dịch chéo](https://clang.llvm.org/docs/CrossCompilation.html#target-triple)).

Ví dụ, chúng ta có thể xem clang cho rằng target triple hiện tại của chúng ta là gì:

```
$ clang --version | grep Target
Target: x86_64-unknown-linux-gnu
```

Khi chạy lệnh này, kết quả trên máy của bạn có thể khác, vì có thể bạn đang dùng kiến trúc hoặc hệ điều hành khác với tôi.

May mắn là chúng ta không cần phải viết cứng (hard-code) target triple để nhắm đến máy hiện tại. LLVM cung cấp `sys::getDefaultTargetTriple`, hàm này trả về target triple của máy hiện tại.

```c++
auto TargetTriple = sys::getDefaultTargetTriple();
```

LLVM không bắt buộc chúng ta phải liên kết toàn bộ chức năng của các kiến trúc đích. Ví dụ, nếu chỉ dùng JIT, chúng ta không cần đến các bộ in mã hợp ngữ (assembly printer). Tương tự, nếu chỉ nhắm đến một số kiến trúc nhất định, chúng ta có thể chỉ liên kết chức năng dành cho những kiến trúc đó.

Trong ví dụ này, chúng ta sẽ khởi tạo tất cả các kiến trúc đích để phát sinh mã đối tượng.

```c++
InitializeAllTargetInfos();
InitializeAllTargets();
InitializeAllTargetMCs();
InitializeAllAsmParsers();
InitializeAllAsmPrinters();
```

Bây giờ chúng ta có thể dùng target triple để lấy một `Target`:

```c++
std::string Error;
auto Target = TargetRegistry::lookupTarget(TargetTriple, Error);

// In ra lỗi và thoát nếu không tìm thấy kiến trúc đích được yêu cầu.
// Điều này thường xảy ra khi chúng ta quên khởi tạo
// TargetRegistry hoặc target triple không hợp lệ.
if (!Target) {
  errs() << Error;
  return 1;
}
```

## Target Machine

Chúng ta cũng cần một `TargetMachine`. Lớp này cung cấp mô tả đầy đủ về máy mà chúng ta đang nhắm đến. Nếu muốn nhắm đến một tính năng cụ thể (chẳng hạn SSE) hoặc một CPU cụ thể (chẳng hạn Sandylake của Intel), chúng ta sẽ chỉ định ngay tại bước này.

Để xem LLVM biết những tính năng và CPU nào, chúng ta có thể dùng `llc`. Ví dụ, hãy xem với x86:

```
$ llvm-as < /dev/null | llc -march=x86 -mattr=help
Available CPUs for this target:

  amdfam10      - Select the amdfam10 processor.
  athlon        - Select the athlon processor.
  athlon-4      - Select the athlon-4 processor.
  ...

Available features for this target:

  16bit-mode            - 16-bit mode (i8086).
  32bit-mode            - 32-bit mode (80386).
  3dnow                 - Enable 3DNow! instructions.
  3dnowa                - Enable 3DNow! Athlon instructions.
  ...
```

Trong ví dụ của chúng ta, chúng ta sẽ dùng CPU chung (generic) mà không kèm thêm tính năng hay tùy chọn đích nào.

```c++
auto CPU = "generic";
auto Features = "";

TargetOptions opt;
auto TargetMachine = Target->createTargetMachine(TargetTriple, CPU, Features, opt, Reloc::PIC_);
```

## Cấu hình Module

Giờ chúng ta đã sẵn sàng cấu hình module, chỉ định kiến trúc đích và bố cục dữ liệu (data layout). Việc này không thực sự bắt buộc, nhưng [hướng dẫn về hiệu năng frontend](../../Frontend/PerformanceTips.md) khuyến nghị nên làm. Các phép tối ưu hóa sẽ hiệu quả hơn khi biết về kiến trúc đích và bố cục dữ liệu.

```c++
TheModule->setDataLayout(TargetMachine->getTargetTriple().computeDataLayout());
TheModule->setTargetTriple(TargetTriple);
```

## Phát sinh mã đối tượng

Chúng ta đã sẵn sàng phát sinh mã đối tượng! Hãy xác định nơi chúng ta muốn ghi tệp ra:

```c++
auto Filename = "output.o";
std::error_code EC;
raw_fd_ostream dest(Filename, EC, sys::fs::OF_None);

if (EC) {
  errs() << "Could not open file: " << EC.message();
  return 1;
}
```

Cuối cùng, chúng ta định nghĩa một pass phát sinh mã đối tượng, rồi chạy pass đó:

```c++
legacy::PassManager pass;
auto FileType = CodeGenFileType::ObjectFile;

if (TargetMachine->addPassesToEmitFile(pass, dest, nullptr, FileType)) {
  errs() << "TargetMachine can't emit a file of this type";
  return 1;
}

pass.run(*TheModule);
dest.flush();
```

## Kết hợp tất cả lại

Nó có hoạt động không? Hãy thử xem. Chúng ta cần biên dịch mã nguồn, nhưng lưu ý rằng các đối số truyền cho `llvm-config` khác với các chương trước.

```
$ clang++ -g -O3 toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs all` -o toy
```

Hãy chạy chương trình và định nghĩa một hàm `average` đơn giản. Nhấn Ctrl-D khi bạn đã xong.

```
$ ./toy
ready> def average(x y) (x + y) * 0.5;
^D
Wrote output.o
```

Chúng ta đã có một tệp đối tượng! Để kiểm tra, hãy viết một chương trình đơn giản và liên kết nó với tệp đầu ra của chúng ta. Đây là mã nguồn:

```c++
#include <iostream>

extern "C" {
    double average(double, double);
}

int main() {
    std::cout << "average of 3.0 and 4.0: " << average(3.0, 4.0) << std::endl;
}
```

Chúng ta liên kết chương trình với output.o và kiểm tra xem kết quả có đúng như mong đợi không:

```
$ clang++ main.cpp output.o -o main
$ ./main
average of 3.0 and 4.0: 3.5
```

## Mã nguồn đầy đủ

```{literalinclude} ../../../examples/Kaleidoscope/Chapter8/toy.cpp
:language: c++
```

[Tiếp theo: Thêm thông tin gỡ lỗi](LangImpl09.md)
