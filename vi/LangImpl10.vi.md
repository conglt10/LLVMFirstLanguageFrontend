# Kaleidoscope: Lời kết và một vài mẹo hữu ích khác về LLVM

## Kết thúc hướng dẫn

Chào mừng bạn đến với chương cuối cùng của loạt hướng dẫn "[Triển khai một ngôn ngữ với LLVM](index.md)". Trong suốt loạt hướng dẫn này, chúng ta đã phát triển ngôn ngữ Kaleidoscope nhỏ bé của mình từ một món đồ chơi vô dụng thành một món đồ chơi khá thú vị (nhưng có lẽ vẫn vô dụng). :)

Thật thú vị khi nhìn lại chặng đường chúng ta đã đi, và lượng code cần thiết lại ít đến nhường nào. Chúng ta đã xây dựng toàn bộ lexer, parser, AST, bộ sinh mã (code generator), một vòng lặp thực thi tương tác (có cả JIT!), và phát sinh thông tin gỡ lỗi (debug information) trong các tệp thực thi độc lập — tất cả chỉ với chưa đến 1000 dòng code (không tính chú thích và dòng trống).

Ngôn ngữ nhỏ bé của chúng ta hỗ trợ một vài tính năng thú vị: nó hỗ trợ các toán tử hai ngôi và một ngôi do người dùng định nghĩa, sử dụng biên dịch JIT để đánh giá biểu thức ngay lập tức, và hỗ trợ một số cấu trúc điều khiển luồng với việc xây dựng dạng SSA.

Một phần mục đích của loạt hướng dẫn này là cho bạn thấy việc định nghĩa, xây dựng và thử nghiệm với ngôn ngữ có thể dễ dàng và thú vị đến mức nào. Xây dựng một trình biên dịch không nhất thiết phải là một quá trình đáng sợ hay bí ẩn! Giờ đây khi bạn đã thấy một số điều cơ bản, tôi rất khuyến khích bạn lấy code này và tự mày mò với nó. Ví dụ, hãy thử thêm:

- **biến toàn cục (global variables)** - Mặc dù giá trị của biến toàn cục trong kỹ nghệ phần mềm hiện đại còn gây tranh cãi, chúng thường hữu ích khi viết nhanh những đoạn code nhỏ kiểu "chữa cháy", như chính trình biên dịch Kaleidoscope. May mắn thay, cấu trúc hiện tại của chúng ta giúp việc thêm biến toàn cục trở nên rất dễ dàng: chỉ cần để phần tra cứu giá trị kiểm tra xem một biến chưa được phân giải có nằm trong bảng ký hiệu biến toàn cục hay không trước khi từ chối nó. Để tạo một biến toàn cục mới, hãy tạo một thể hiện của lớp `GlobalVariable` trong LLVM.
- **biến có kiểu (typed variables)** - Kaleidoscope hiện chỉ hỗ trợ các biến kiểu double. Điều này mang lại cho ngôn ngữ một sự thanh lịch rất đẹp, bởi vì chỉ hỗ trợ một kiểu nghĩa là bạn không bao giờ phải chỉ định kiểu. Các ngôn ngữ khác nhau có những cách xử lý vấn đề này khác nhau. Cách dễ nhất là yêu cầu người dùng chỉ định kiểu cho mọi định nghĩa biến, và ghi lại kiểu của biến trong bảng ký hiệu cùng với `Value*` của nó.
- **mảng, struct, vector, v.v.** - Một khi đã thêm kiểu, bạn có thể bắt đầu mở rộng hệ thống kiểu theo đủ mọi cách thú vị. Mảng đơn giản rất dễ triển khai và khá hữu ích cho nhiều ứng dụng khác nhau. Việc thêm chúng chủ yếu là một bài tập để tìm hiểu cách hoạt động của lệnh [getelementptr](../../LangRef.md#getelementptr-instruction) trong LLVM: lệnh này độc đáo/khác thường đến mức nó [có hẳn một trang FAQ riêng](../../GetElementPtr.md)!
- **runtime chuẩn (standard runtime)** - Ngôn ngữ hiện tại của chúng ta cho phép người dùng truy cập các hàm bên ngoài tùy ý, và chúng ta dùng nó cho những thứ như "printd" và "putchard". Khi bạn mở rộng ngôn ngữ để thêm các cấu trúc cấp cao hơn, thường thì các cấu trúc này hợp lý nhất khi được hạ cấp (lower) thành các lời gọi tới một runtime do ngôn ngữ cung cấp. Ví dụ, nếu bạn thêm bảng băm (hash table) vào ngôn ngữ, có lẽ sẽ hợp lý hơn nếu đưa các thủ tục xử lý vào một runtime thay vì inline toàn bộ chúng.
- **quản lý bộ nhớ** - Hiện tại chúng ta chỉ có thể truy cập stack trong Kaleidoscope. Sẽ rất hữu ích nếu có thể cấp phát bộ nhớ heap, hoặc bằng các lời gọi tới giao diện malloc/free chuẩn của libc, hoặc bằng một bộ thu gom rác (garbage collector). Nếu bạn muốn sử dụng thu gom rác, lưu ý rằng LLVM hỗ trợ đầy đủ [Thu gom rác chính xác (Accurate Garbage Collection)](../../GarbageCollection.md), bao gồm cả các thuật toán di chuyển đối tượng và cần quét/cập nhật stack.
- **hỗ trợ xử lý ngoại lệ** - LLVM hỗ trợ sinh [ngoại lệ chi phí bằng không (zero cost exceptions)](../../ExceptionHandling.md) có khả năng tương tác với code được biên dịch từ các ngôn ngữ khác. Bạn cũng có thể sinh code bằng cách ngầm định cho mọi hàm trả về một giá trị lỗi và kiểm tra giá trị đó. Bạn cũng có thể sử dụng setjmp/longjmp một cách tường minh. Có rất nhiều hướng khác nhau để đi ở đây.
- **hướng đối tượng, generic, truy cập cơ sở dữ liệu, số phức, lập trình hình học, ...** - Thực sự là không có giới hạn nào cho những tính năng "điên rồ" mà bạn có thể thêm vào ngôn ngữ.
- **các lĩnh vực khác thường** - Chúng ta đã nói về việc áp dụng LLVM vào một lĩnh vực mà nhiều người quan tâm: xây dựng trình biên dịch cho một ngôn ngữ cụ thể. Tuy nhiên, có rất nhiều lĩnh vực khác có thể sử dụng công nghệ trình biên dịch mà thường không được nghĩ tới. Ví dụ, LLVM đã được dùng để triển khai tăng tốc đồ họa OpenGL, dịch code C++ sang ActionScript, và nhiều điều thú vị, thông minh khác. Biết đâu bạn sẽ là người đầu tiên biên dịch JIT một trình thông dịch biểu thức chính quy (regular expression) thành mã máy với LLVM?

Hãy vui vẻ — hãy thử làm điều gì đó điên rồ và khác thường. Xây dựng một ngôn ngữ giống như mọi người vẫn luôn làm sẽ kém thú vị hơn nhiều so với việc thử một thứ hơi điên rồ hoặc lập dị và xem kết quả ra sao. Nếu bạn gặp khó khăn hoặc muốn trao đổi về nó, hãy đăng bài lên [diễn đàn LLVM](https://discourse.llvm.org): ở đó có rất nhiều người quan tâm đến ngôn ngữ lập trình và thường sẵn lòng giúp đỡ.

Trước khi kết thúc loạt hướng dẫn này, tôi muốn nói về một số "mẹo và thủ thuật" khi sinh LLVM IR. Đây là những điều tinh tế hơn, có thể không hiển nhiên, nhưng rất hữu ích nếu bạn muốn tận dụng các khả năng của LLVM.

## Các đặc tính của LLVM IR

Chúng tôi thường nhận được một vài câu hỏi phổ biến về code ở dạng LLVM IR — hãy cùng giải quyết chúng ngay bây giờ nhé?

### Tính độc lập với nền tảng đích

Kaleidoscope là một ví dụ về "ngôn ngữ khả chuyển" (portable language): bất kỳ chương trình nào viết bằng Kaleidoscope cũng sẽ hoạt động giống nhau trên mọi nền tảng đích (target) mà nó chạy. Nhiều ngôn ngữ khác cũng có đặc tính này, ví dụ lisp, java, haskell, javascript, python, v.v. (lưu ý rằng mặc dù các ngôn ngữ này khả chuyển, không phải tất cả thư viện của chúng đều như vậy).

Một điểm hay của LLVM là nó thường có khả năng bảo toàn tính độc lập với nền tảng đích trong IR: bạn có thể lấy LLVM IR của một chương trình được biên dịch bởi Kaleidoscope và chạy nó trên bất kỳ nền tảng đích nào mà LLVM hỗ trợ, thậm chí phát sinh code C và biên dịch nó trên những nền tảng mà LLVM không hỗ trợ gốc. Bạn có thể dễ dàng nhận thấy trình biên dịch Kaleidoscope sinh ra code độc lập với nền tảng đích, bởi vì nó không bao giờ truy vấn bất kỳ thông tin đặc thù nào của nền tảng đích khi sinh code.

Việc LLVM cung cấp một dạng biểu diễn code gọn nhẹ và độc lập với nền tảng đích khiến rất nhiều người hào hứng. Đáng tiếc là những người này thường nghĩ đến C hoặc một ngôn ngữ thuộc họ C khi họ đặt câu hỏi về tính khả chuyển của ngôn ngữ. Tôi nói "đáng tiếc" bởi vì thực sự không có cách nào để làm cho code C (ở dạng tổng quát đầy đủ) trở nên khả chuyển, ngoài việc phân phối chính mã nguồn (và tất nhiên, nói chung mã nguồn C cũng không thực sự khả chuyển — bạn đã bao giờ port một ứng dụng thật cũ từ 32-bit lên 64-bit chưa?).

Vấn đề với C (một lần nữa, ở dạng tổng quát đầy đủ của nó) là nó chứa đầy những giả định đặc thù cho nền tảng đích. Lấy một ví dụ đơn giản, bộ tiền xử lý (preprocessor) thường phá hủy tính độc lập với nền tảng đích của code khi nó xử lý văn bản đầu vào:

```c
#ifdef __i386__
  int X = 1;
#else
  int X = 42;
#endif
```

Mặc dù có thể thiết kế những giải pháp ngày càng phức tạp cho các vấn đề kiểu này, nhưng không thể giải quyết chúng một cách tổng quát đầy đủ theo cách tốt hơn việc phân phối chính mã nguồn.

Dù vậy, có những tập con thú vị của C có thể được làm cho khả chuyển. Nếu bạn sẵn sàng cố định kích thước của các kiểu nguyên thủy (chẳng hạn int = 32-bit và long = 64-bit), không quan tâm đến khả năng tương thích ABI với các tệp nhị phân hiện có, và sẵn sàng từ bỏ một vài tính năng nhỏ khác, bạn có thể có được code khả chuyển. Điều này có thể hợp lý cho các lĩnh vực chuyên biệt, chẳng hạn như một ngôn ngữ chạy bên trong kernel.

### Đảm bảo an toàn

Nhiều ngôn ngữ kể trên cũng là các ngôn ngữ "an toàn": một chương trình viết bằng Java không thể làm hỏng không gian địa chỉ của nó và khiến tiến trình bị crash (giả sử JVM không có lỗi). An toàn là một đặc tính thú vị, đòi hỏi sự kết hợp giữa thiết kế ngôn ngữ, hỗ trợ từ runtime, và thường cả hỗ trợ từ hệ điều hành.

Hoàn toàn có thể triển khai một ngôn ngữ an toàn trên LLVM, nhưng bản thân LLVM IR không đảm bảo an toàn. LLVM IR cho phép ép kiểu con trỏ không an toàn, lỗi sử dụng sau khi giải phóng (use after free), tràn bộ đệm (buffer over-run), và nhiều vấn đề khác. Tính an toàn cần được triển khai như một lớp nằm trên LLVM và, thật tiện lợi, đã có một số nhóm nghiên cứu về vấn đề này. Hãy hỏi trên [diễn đàn LLVM](https://discourse.llvm.org) nếu bạn muốn biết thêm chi tiết.

### Tối ưu hóa đặc thù cho ngôn ngữ

Một điều về LLVM khiến nhiều người không thích là nó không giải quyết mọi vấn đề trên đời chỉ trong một hệ thống. Một lời phàn nàn cụ thể là mọi người cho rằng LLVM không có khả năng thực hiện tối ưu hóa cấp cao đặc thù cho ngôn ngữ: LLVM "làm mất quá nhiều thông tin". Dưới đây là một vài nhận xét về vấn đề này:

Thứ nhất, bạn nói đúng rằng LLVM có làm mất thông tin. Ví dụ, tại thời điểm viết bài này, không có cách nào để phân biệt trong LLVM IR liệu một giá trị SSA đến từ kiểu "int" hay "long" của C trên một máy ILP32 (ngoại trừ thông tin gỡ lỗi). Cả hai đều được biên dịch thành giá trị 'i32' và thông tin về nguồn gốc của nó bị mất. Vấn đề tổng quát hơn ở đây là hệ thống kiểu của LLVM sử dụng "tương đương cấu trúc" (structural equivalence) thay vì "tương đương tên" (name equivalence). Một trường hợp khác khiến mọi người bất ngờ là khi bạn có hai kiểu trong một ngôn ngữ cấp cao có cùng cấu trúc (ví dụ: hai struct khác nhau cùng có một trường int duy nhất): các kiểu này sẽ được biên dịch thành một kiểu LLVM duy nhất và sẽ không thể biết được nó đến từ đâu.

Thứ hai, mặc dù LLVM có làm mất thông tin, LLVM không phải là một mục tiêu cố định: chúng tôi liên tục nâng cấp và cải thiện nó theo nhiều cách khác nhau. Ngoài việc thêm các tính năng mới (LLVM không phải lúc nào cũng hỗ trợ ngoại lệ hay thông tin gỡ lỗi), chúng tôi cũng mở rộng IR để nắm bắt những thông tin quan trọng cho việc tối ưu hóa (ví dụ: một đối số được mở rộng dấu (sign extend) hay mở rộng bằng số không (zero extend), thông tin về bí danh con trỏ (pointer aliasing), v.v.). Nhiều cải tiến xuất phát từ người dùng: mọi người muốn LLVM có một tính năng cụ thể nào đó, nên họ bắt tay vào mở rộng nó.

Thứ ba, việc thêm các tối ưu hóa đặc thù cho ngôn ngữ là *khả thi và dễ dàng*, và bạn có nhiều lựa chọn về cách thực hiện. Lấy một ví dụ đơn giản, rất dễ để thêm các pass tối ưu hóa đặc thù cho ngôn ngữ "hiểu biết" về code được biên dịch từ ngôn ngữ đó. Trong trường hợp họ ngôn ngữ C, có một pass tối ưu hóa "hiểu biết" về các hàm thư viện chuẩn của C. Nếu bạn gọi "exit(0)" trong main(), nó biết rằng có thể tối ưu hóa an toàn lời gọi đó thành "return 0;" vì C đặc tả rõ hàm 'exit' làm gì.

Ngoài hiểu biết đơn giản về thư viện, còn có thể nhúng nhiều loại thông tin đặc thù cho ngôn ngữ khác vào LLVM IR. Nếu bạn có một nhu cầu cụ thể và gặp bế tắc, hãy đưa chủ đề đó lên danh sách thư llvm-dev. Trong trường hợp xấu nhất, bạn luôn có thể coi LLVM như một "bộ sinh mã đơn thuần" và triển khai các tối ưu hóa cấp cao mà bạn mong muốn ngay trong front-end, trên AST đặc thù cho ngôn ngữ.

## Mẹo và thủ thuật

Có rất nhiều mẹo và thủ thuật hữu ích mà bạn sẽ biết được sau khi làm việc với/trên LLVM, những điều không hiển nhiên ngay từ cái nhìn đầu tiên. Thay vì để mọi người tự khám phá lại chúng, phần này sẽ đề cập đến một số vấn đề như vậy.

### Triển khai offsetof/sizeof khả chuyển

Một điều thú vị nảy sinh khi bạn cố gắng giữ cho code do trình biên dịch sinh ra "độc lập với nền tảng đích", đó là bạn thường cần biết kích thước của một kiểu LLVM nào đó hoặc offset của một trường nào đó trong một struct của llvm. Ví dụ, bạn có thể cần truyền kích thước của một kiểu vào một hàm cấp phát bộ nhớ.

Đáng tiếc là những giá trị này có thể khác nhau rất nhiều giữa các nền tảng đích: ví dụ, độ rộng của một con trỏ hiển nhiên là đặc thù cho từng nền tảng đích. Tuy nhiên, có một [cách thông minh để sử dụng lệnh getelementptr](http://nondot.org/sabre/LLVMNotes/SizeOf-OffsetOf-VariableSizedStructs.txt) cho phép bạn tính toán điều này theo cách khả chuyển.

### Stack frame được thu gom rác

Một số ngôn ngữ muốn quản lý stack frame của chúng một cách tường minh, thường là để chúng được thu gom rác hoặc để dễ dàng triển khai closure. Thường có những cách tốt hơn để triển khai các tính năng này thay vì dùng stack frame tường minh, nhưng [LLVM vẫn hỗ trợ chúng](http://nondot.org/sabre/LLVMNotes/ExplicitlyManagedStackFrames.txt) nếu bạn muốn. Điều này đòi hỏi front-end của bạn phải chuyển đổi code sang [Phong cách truyền continuation (Continuation Passing Style)](http://en.wikipedia.org/wiki/Continuation-passing_style) và sử dụng lời gọi đuôi (tail call) — thứ mà LLVM cũng hỗ trợ.
