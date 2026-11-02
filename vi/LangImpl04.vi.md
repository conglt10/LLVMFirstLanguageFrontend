# Kaleidoscope: Thêm hỗ trợ JIT và bộ tối ưu hóa

## Giới thiệu Chương 4

Chào mừng bạn đến với Chương 4 của hướng dẫn "[Hiện thực một ngôn ngữ với LLVM](index.md)". Các Chương 1-3 đã mô tả cách hiện thực một ngôn ngữ đơn giản và bổ sung khả năng sinh mã LLVM IR. Chương này giới thiệu hai kỹ thuật mới: thêm hỗ trợ bộ tối ưu hóa (optimizer) cho ngôn ngữ của bạn, và thêm hỗ trợ trình biên dịch JIT. Những bổ sung này sẽ cho thấy cách tạo ra mã đẹp và hiệu quả cho ngôn ngữ Kaleidoscope.

## Gập hằng số đơn giản

Phần minh họa của Chương 3 khá thanh lịch và dễ mở rộng. Đáng tiếc là nó không sinh ra mã thật sự tốt. Tuy vậy, IRBuilder vẫn mang lại cho chúng ta những tối ưu hóa hiển nhiên khi biên dịch mã đơn giản:

```
ready> def test(x) 1+2+x;
Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double 3.000000e+00, %x
        ret double %addtmp
}
```

Đoạn mã này không phải là bản chép nguyên văn từ AST được xây dựng khi phân tích cú pháp đầu vào. Nếu chép nguyên văn, nó sẽ trông như sau:

```
ready> def test(x) 1+2+x;
Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double 2.000000e+00, 1.000000e+00
        %addtmp1 = fadd double %addtmp, %x
        ret double %addtmp1
}
```

Gập hằng số (constant folding), như ta thấy ở trên, là một tối ưu hóa rất phổ biến và rất quan trọng: quan trọng đến mức nhiều người hiện thực ngôn ngữ cài đặt sẵn khả năng gập hằng số ngay trong biểu diễn AST của họ.

Với LLVM, bạn không cần hỗ trợ này trong AST. Vì mọi lệnh gọi để xây dựng LLVM IR đều đi qua LLVM IR builder, nên chính builder sẽ kiểm tra xem có cơ hội gập hằng số hay không mỗi khi bạn gọi nó. Nếu có, nó chỉ việc gập hằng số và trả về hằng số đó thay vì tạo ra một lệnh (instruction).

Chà, dễ thật :). Trên thực tế, chúng tôi khuyên bạn luôn dùng `IRBuilder` khi sinh mã theo cách này. Nó không gây ra "chi phí cú pháp" nào khi sử dụng (bạn không phải làm xấu trình biên dịch của mình bằng việc kiểm tra hằng số ở khắp nơi) và trong một số trường hợp, nó có thể giảm đáng kể lượng LLVM IR được sinh ra (đặc biệt với những ngôn ngữ có bộ tiền xử lý macro hoặc dùng nhiều hằng số).

Mặt khác, `IRBuilder` bị giới hạn bởi việc nó thực hiện mọi phân tích ngay tại chỗ, trong lúc mã đang được xây dựng. Hãy xem một ví dụ phức tạp hơn một chút:

```
ready> def test(x) (1+2+x)*(x+(1+2));
ready> Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double 3.000000e+00, %x
        %addtmp1 = fadd double %x, 3.000000e+00
        %multmp = fmul double %addtmp, %addtmp1
        ret double %multmp
}
```

Trong trường hợp này, vế trái (LHS) và vế phải (RHS) của phép nhân có cùng giá trị. Ta thực sự muốn mã sinh ra là "`tmp = x+3; result = tmp*tmp;`" thay vì tính "`x+3`" hai lần.

Đáng tiếc là không có phân tích cục bộ nào có thể phát hiện và sửa được điều này. Việc đó cần đến hai phép biến đổi: tái kết hợp (reassociation) các biểu thức (để các phép cộng trở nên giống hệt nhau về mặt từ vựng) và Loại bỏ biểu thức con chung (Common Subexpression Elimination - CSE) để xóa lệnh cộng dư thừa. May mắn thay, LLVM cung cấp rất nhiều tối ưu hóa mà bạn có thể sử dụng, dưới dạng các "pass".

## Các pass tối ưu hóa của LLVM

LLVM cung cấp nhiều pass tối ưu hóa, mỗi pass làm những việc khác nhau và có những đánh đổi khác nhau. Không giống các hệ thống khác, LLVM không giữ quan niệm sai lầm rằng chỉ một bộ tối ưu hóa là phù hợp cho mọi ngôn ngữ và mọi tình huống. LLVM cho phép người hiện thực trình biên dịch toàn quyền quyết định dùng những tối ưu hóa nào, theo thứ tự nào và trong tình huống nào.

Lấy một ví dụ cụ thể, LLVM hỗ trợ các pass "toàn module" (whole module), vốn xem xét một khối mã lớn nhất có thể (thường là cả một tệp, nhưng nếu chạy lúc liên kết thì có thể là một phần đáng kể của toàn bộ chương trình). LLVM cũng hỗ trợ và cung cấp sẵn các pass "theo từng hàm" (per-function), chỉ thao tác trên một hàm tại một thời điểm mà không xem xét các hàm khác. Để biết thêm thông tin về các pass và cách chúng được chạy, hãy xem tài liệu [How to Write a Pass](../../WritingAnLLVMPass.md) và [List of LLVM Passes](../../Passes.md).

Với Kaleidoscope, hiện tại chúng ta đang sinh các hàm ngay lập tức, từng hàm một, khi người dùng gõ chúng vào. Trong bối cảnh này, ta không nhắm đến trải nghiệm tối ưu hóa tột bậc, nhưng vẫn muốn nắm bắt những tối ưu dễ và nhanh khi có thể. Vì vậy, ta sẽ chọn chạy một vài tối ưu hóa theo từng hàm khi người dùng gõ hàm vào. Nếu muốn tạo một "trình biên dịch Kaleidoscope tĩnh", ta sẽ dùng đúng đoạn mã hiện có, chỉ khác là trì hoãn việc chạy bộ tối ưu hóa cho đến khi toàn bộ tệp đã được phân tích cú pháp xong.

Ngoài sự phân biệt giữa pass theo hàm và pass theo module, các pass còn có thể được chia thành pass biến đổi (transform pass) và pass phân tích (analysis pass). Pass biến đổi làm thay đổi IR, còn pass phân tích tính toán thông tin mà các pass khác có thể sử dụng. Để thêm một pass biến đổi, mọi pass phân tích mà nó phụ thuộc vào phải được đăng ký từ trước.

Để bắt đầu tối ưu hóa theo từng hàm, ta cần thiết lập một [FunctionPassManager](../../WritingAnLLVMPass.md#what-passmanager-does) để chứa và tổ chức các tối ưu hóa LLVM mà ta muốn chạy. Khi đã có nó, ta có thể thêm một tập các tối ưu hóa cần chạy. Ta sẽ cần một FunctionPassManager mới cho mỗi module muốn tối ưu hóa, vì vậy ta sẽ bổ sung vào một hàm đã tạo ở chương trước (`InitializeModule()`):

```c++
void InitializeModuleAndManagers(void) {
  // Mở một context và module mới.
  TheContext = std::make_unique<LLVMContext>();
  TheModule = std::make_unique<Module>("KaleidoscopeJIT", *TheContext);
  TheModule->setDataLayout(TheJIT->getDataLayout());

  // Tạo một builder mới cho module.
  Builder = std::make_unique<IRBuilder<>>(*TheContext);

  // Tạo các pass manager và analysis manager mới.
  TheFPM = std::make_unique<FunctionPassManager>();
  TheLAM = std::make_unique<LoopAnalysisManager>();
  TheFAM = std::make_unique<FunctionAnalysisManager>();
  TheCGAM = std::make_unique<CGSCCAnalysisManager>();
  TheMAM = std::make_unique<ModuleAnalysisManager>();
  ThePIC = std::make_unique<PassInstrumentationCallbacks>();
  TheSI = std::make_unique<StandardInstrumentations>(*TheContext,
                                                    /*DebugLogging*/ true);
  TheSI->registerCallbacks(*ThePIC, TheMAM.get());
  ...
```

Bên cạnh phần khởi tạo, trong mã còn có dòng `TheModule->setDataLayout(TheJIT->getDataLayout());`. Chúng ta sẽ bàn về nó ở phần sau của chương này.

Sau khi khởi tạo module toàn cục `TheModule` và FunctionPassManager, ta cần khởi tạo các phần khác của framework. Bốn AnalysisManager cho phép ta thêm các pass phân tích chạy trên bốn cấp của hệ phân cấp IR. PassInstrumentationCallbacks và StandardInstrumentations là bắt buộc đối với framework đo đạc pass (pass instrumentation), cho phép nhà phát triển tùy chỉnh những gì xảy ra giữa các pass.

Khi các manager này đã được thiết lập, ta dùng một loạt lời gọi "addPass" để thêm một số pass biến đổi của LLVM:

```c++
// Thêm các pass biến đổi.
// Thực hiện các tối ưu hóa "peephole" đơn giản và tối ưu hóa thao tác bit.
TheFPM->addPass(InstCombinePass());
// Tái kết hợp các biểu thức.
TheFPM->addPass(ReassociatePass());
// Loại bỏ các biểu thức con chung.
TheFPM->addPass(GVNPass());
// Đơn giản hóa đồ thị luồng điều khiển (xóa các khối không thể tới được, v.v.).
TheFPM->addPass(SimplifyCFGPass());
```

Trong trường hợp này, ta chọn thêm bốn pass tối ưu hóa. Các pass được chọn ở đây là một bộ tối ưu hóa "dọn dẹp" khá tiêu chuẩn, hữu ích cho nhiều loại mã khác nhau. Tôi sẽ không đi sâu vào việc chúng làm gì, nhưng tin tôi đi, chúng là một điểm khởi đầu tốt :).

Tiếp theo, ta đăng ký các pass phân tích được các pass biến đổi sử dụng.

```c++
  // Đăng ký các pass phân tích được dùng trong các pass biến đổi này.
  PassBuilder PB(*TheContext);
  PB.registerModuleAnalyses(*TheMAM);
  PB.registerFunctionAnalyses(*TheFAM);
  PB.crossRegisterProxies(*TheLAM, *TheFAM, *TheCGAM, *TheMAM);
}
```

Khi PassManager đã được thiết lập, ta cần sử dụng nó. Ta làm điều này bằng cách chạy nó sau khi hàm mới được xây dựng xong (trong `FunctionAST::codegen()`), nhưng trước khi hàm được trả về cho phía gọi:

```c++
if (Value *RetVal = Body->codegen()) {
  // Hoàn tất hàm.
  Builder.CreateRet(RetVal);

  // Kiểm tra mã được sinh ra, đảm bảo tính nhất quán.
  verifyFunction(*TheFunction);

  // Tối ưu hóa hàm.
  TheFPM->run(*TheFunction, *TheFAM);

  return TheFunction;
}
```

Như bạn thấy, việc này khá đơn giản. `FunctionPassManager` tối ưu hóa và cập nhật trực tiếp `Function*` của LLVM, (hy vọng là) cải thiện phần thân của nó. Với thay đổi này, ta có thể thử lại bài kiểm tra ở trên:

```
ready> def test(x) (1+2+x)*(x+(1+2));
ready> Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double %x, 3.000000e+00
        %multmp = fmul double %addtmp, %addtmp
        ret double %multmp
}
```

Đúng như kỳ vọng, giờ đây ta đã có mã được tối ưu hóa gọn gàng, tiết kiệm được một lệnh cộng số thực dấu phẩy động mỗi lần thực thi hàm này.

LLVM cung cấp rất nhiều tối ưu hóa có thể dùng trong những hoàn cảnh nhất định. Có sẵn một số [tài liệu về các pass](../../Passes.md), nhưng chúng chưa thật sự đầy đủ. Một nguồn ý tưởng tốt khác là xem các pass mà `Clang` chạy để bắt đầu. Công cụ "`opt`" cho phép bạn thử nghiệm các pass từ dòng lệnh, nhờ vậy bạn có thể xem chúng có tác dụng gì hay không.

Giờ đây front-end của chúng ta đã sinh ra mã khá ổn, hãy cùng bàn về việc thực thi nó!

## Thêm trình biên dịch JIT

Mã ở dạng LLVM IR có thể được xử lý bởi rất nhiều công cụ khác nhau. Ví dụ, bạn có thể chạy các tối ưu hóa trên nó (như ta đã làm ở trên), có thể xuất nó ra dưới dạng văn bản hoặc nhị phân, có thể biên dịch mã thành tệp hợp ngữ (.s) cho một kiến trúc đích nào đó, hoặc có thể biên dịch JIT cho nó. Điều hay ở biểu diễn LLVM IR là nó đóng vai trò "đồng tiền chung" giữa nhiều phần khác nhau của trình biên dịch.

Trong phần này, chúng ta sẽ thêm hỗ trợ trình biên dịch JIT cho trình thông dịch của mình. Ý tưởng cơ bản cho Kaleidoscope là người dùng vẫn nhập thân hàm như hiện tại, nhưng các biểu thức cấp cao nhất (top-level expression) họ gõ vào sẽ được đánh giá ngay lập tức. Ví dụ, nếu họ gõ "1 + 2;", ta nên đánh giá và in ra 3. Nếu họ định nghĩa một hàm, họ phải có thể gọi hàm đó từ dòng lệnh.

Để làm được điều này, trước tiên ta chuẩn bị môi trường để tạo mã cho kiến trúc đích gốc (native target) hiện tại, đồng thời khai báo và khởi tạo JIT. Việc này được thực hiện bằng cách gọi một số hàm `InitializeNativeTarget*`, thêm biến toàn cục `TheJIT` và khởi tạo nó trong `main`:

```c++
static std::unique_ptr<KaleidoscopeJIT> TheJIT;
...
int main() {
  InitializeNativeTarget();
  InitializeNativeTargetAsmPrinter();
  InitializeNativeTargetAsmParser();

  // Cài đặt các toán tử hai ngôi tiêu chuẩn.
  // 1 là độ ưu tiên thấp nhất.
  BinopPrecedence['<'] = 10;
  BinopPrecedence['+'] = 20;
  BinopPrecedence['-'] = 20;
  BinopPrecedence['*'] = 40; // cao nhất.

  // Nạp sẵn token đầu tiên.
  fprintf(stderr, "ready> ");
  getNextToken();

  TheJIT = std::make_unique<KaleidoscopeJIT>();

  // Bây giờ chạy "vòng lặp thông dịch" chính.
  MainLoop();

  return 0;
}
```

Ta cũng cần thiết lập data layout cho JIT. Giờ đây `TheJIT` đã tồn tại, ta có thể thêm lời gọi `setDataLayout` vào `InitializeModuleAndManagers`:

```c++
void InitializeModuleAndManagers(void) {
  // Mở một context và module mới.
  TheContext = std::make_unique<LLVMContext>();
  TheModule = std::make_unique<Module>("KaleidoscopeJIT", *TheContext);
  TheModule->setDataLayout(TheJIT->getDataLayout());
  ...
```

Lớp KaleidoscopeJIT là một JIT đơn giản được xây dựng riêng cho loạt hướng dẫn này, có sẵn trong mã nguồn LLVM tại [llvm-src/examples/Kaleidoscope/include/KaleidoscopeJIT.h](https://github.com/llvm/llvm-project/blob/main/llvm/examples/Kaleidoscope/include/KaleidoscopeJIT.h). Ở các chương sau, chúng ta sẽ tìm hiểu cách nó hoạt động và mở rộng nó với các tính năng mới, nhưng hiện tại ta tạm coi nó là có sẵn. API của nó rất đơn giản: `addModule` thêm một module LLVM IR vào JIT, giúp các hàm của module đó sẵn sàng để thực thi (với bộ nhớ được quản lý bởi một `ResourceTracker`); còn `lookup` cho phép ta tra cứu con trỏ tới mã đã được biên dịch.

Với API đơn giản này, ta có thể sửa đoạn mã phân tích biểu thức cấp cao nhất thành như sau:

```c++
static ExitOnError ExitOnErr;
...
static void HandleTopLevelExpression() {
  // Đánh giá một biểu thức cấp cao nhất thành một hàm ẩn danh.
  if (auto FnAST = ParseTopLevelExpr()) {
    if (FnAST->codegen()) {
      // Tạo một ResourceTracker để theo dõi bộ nhớ JIT cấp phát cho
      // biểu thức ẩn danh -- nhờ đó ta có thể giải phóng nó sau khi thực thi.
      auto RT = TheJIT->getMainJITDylib().createResourceTracker();

      auto TSM = ThreadSafeModule(std::move(TheModule), std::move(TheContext));
      ExitOnErr(TheJIT->addModule(std::move(TSM), RT));
      InitializeModuleAndManagers();

      // Tìm ký hiệu __anon_expr trong JIT.
      auto ExprSymbol = ExitOnErr(TheJIT->lookup("__anon_expr"));

      // Lấy địa chỉ của ký hiệu và ép kiểu về đúng kiểu (không nhận
      // đối số, trả về double) để có thể gọi nó như một hàm gốc.
      double (*FP)() = ExprSymbol.toPtr<double (*)()>();
      fprintf(stderr, "Evaluated to %f\n", FP());

      // Xóa module của biểu thức ẩn danh khỏi JIT.
      ExitOnErr(RT->remove());
    }
```

Nếu phân tích cú pháp và sinh mã thành công, bước tiếp theo là thêm module chứa biểu thức cấp cao nhất vào JIT. Ta làm điều này bằng cách gọi addModule, hàm này kích hoạt việc sinh mã cho mọi hàm trong module và nhận vào một `ResourceTracker` có thể dùng để xóa module khỏi JIT sau này. Một khi module đã được thêm vào JIT thì không thể sửa đổi nó nữa, vì vậy ta cũng mở một module mới để chứa mã tiếp theo bằng cách gọi `InitializeModuleAndManagers()`.

Sau khi đã thêm module vào JIT, ta cần lấy con trỏ tới mã cuối cùng được sinh ra. Ta làm điều này bằng cách gọi phương thức `lookup` của JIT và truyền vào tên của hàm biểu thức cấp cao nhất: `__anon_expr`.

Tiếp theo, ta lấy địa chỉ trong bộ nhớ của hàm `__anon_expr`. Hãy nhớ lại rằng ta biên dịch các biểu thức cấp cao nhất thành một hàm LLVM độc lập, không nhận đối số và trả về giá trị double đã tính được. Vì trình biên dịch JIT của LLVM tuân theo ABI của nền tảng gốc, nên bạn chỉ cần ép kiểu con trỏ kết quả thành con trỏ hàm có kiểu tương ứng rồi gọi trực tiếp. Điều này có nghĩa là không có sự khác biệt nào giữa mã được biên dịch JIT và mã máy gốc được liên kết tĩnh vào ứng dụng của bạn.

Cuối cùng, vì ta không hỗ trợ đánh giá lại các biểu thức cấp cao nhất, ta xóa module khỏi JIT khi đã xong để giải phóng bộ nhớ liên quan. Tuy nhiên, hãy nhớ rằng module mà ta tạo ra vài dòng trước đó (thông qua `InitializeModuleAndManagers`) vẫn đang mở và chờ mã mới được thêm vào.

Chỉ với hai thay đổi này, hãy xem Kaleidoscope hoạt động thế nào!

```
ready> 4+5;
Read top-level expression:
define double @0() {
entry:
  ret double 9.000000e+00
}

Evaluated to 9.000000
```

Có vẻ như về cơ bản nó đã hoạt động. Bản kết xuất của hàm cho thấy "hàm không đối số luôn trả về double" mà ta tổng hợp cho mỗi biểu thức cấp cao nhất được gõ vào. Đây mới chỉ là chức năng rất cơ bản, nhưng liệu ta có thể làm được nhiều hơn không?

```
ready> def testfunc(x y) x + y*2;
Read function definition:
define double @testfunc(double %x, double %y) {
entry:
  %multmp = fmul double %y, 2.000000e+00
  %addtmp = fadd double %multmp, %x
  ret double %addtmp
}

ready> testfunc(4, 10);
Read top-level expression:
define double @1() {
entry:
  %calltmp = call double @testfunc(double 4.000000e+00, double 1.000000e+01)
  ret double %calltmp
}

Evaluated to 24.000000

ready> testfunc(5, 10);
ready> LLVM ERROR: Program used external function 'testfunc' which could not be resolved!
```

Định nghĩa hàm và lời gọi hàm cũng hoạt động, nhưng có gì đó đã sai nghiêm trọng ở dòng cuối cùng. Lời gọi trông có vẻ hợp lệ, vậy chuyện gì đã xảy ra? Có thể bạn đã đoán ra từ API: Module là đơn vị cấp phát của JIT, và testfunc thuộc cùng module với biểu thức ẩn danh. Khi ta xóa module đó khỏi JIT để giải phóng bộ nhớ của biểu thức ẩn danh, ta cũng đã xóa luôn định nghĩa của `testfunc`. Sau đó, khi ta gọi testfunc lần thứ hai, JIT không còn tìm thấy nó nữa.

Cách dễ nhất để khắc phục là đặt biểu thức ẩn danh vào một module tách biệt với các định nghĩa hàm còn lại. JIT sẽ vui vẻ phân giải các lời gọi hàm xuyên qua ranh giới module, miễn là mỗi hàm được gọi đều có prototype và được thêm vào JIT trước khi được gọi. Bằng cách đặt biểu thức ẩn danh vào một module khác, ta có thể xóa nó mà không ảnh hưởng đến các hàm còn lại.

Thực tế, ta sẽ tiến thêm một bước nữa và đặt mỗi hàm vào một module riêng. Làm vậy cho phép ta tận dụng một đặc tính hữu ích của KaleidoscopeJIT, giúp môi trường của ta giống một REPL hơn: các hàm có thể được thêm vào JIT nhiều hơn một lần (khác với một module, nơi mỗi hàm phải có một định nghĩa duy nhất). Khi bạn tra cứu một ký hiệu trong KaleidoscopeJIT, nó sẽ luôn trả về định nghĩa gần đây nhất:

```
ready> def foo(x) x + 1;
Read function definition:
define double @foo(double %x) {
entry:
  %addtmp = fadd double %x, 1.000000e+00
  ret double %addtmp
}

ready> foo(2);
Evaluated to 3.000000

ready> def foo(x) x + 2;
define double @foo(double %x) {
entry:
  %addtmp = fadd double %x, 2.000000e+00
  ret double %addtmp
}

ready> foo(2);
Evaluated to 4.000000
```

Để mỗi hàm có thể nằm trong module riêng của nó, ta cần một cách để sinh lại các khai báo hàm trước đó vào mỗi module mới mà ta mở ra:

```c++
static std::unique_ptr<KaleidoscopeJIT> TheJIT;

...

Function *getFunction(std::string Name) {
  // Trước tiên, kiểm tra xem hàm đã được thêm vào module hiện tại chưa.
  if (auto *F = TheModule->getFunction(Name))
    return F;

  // Nếu chưa, kiểm tra xem có thể sinh mã khai báo từ một prototype
  // sẵn có hay không.
  auto FI = FunctionProtos.find(Name);
  if (FI != FunctionProtos.end())
    return FI->second->codegen();

  // Nếu không có prototype nào, trả về null.
  return nullptr;
}

...

Value *CallExprAST::codegen() {
  // Tra cứu tên trong bảng module toàn cục.
  Function *CalleeF = getFunction(Callee);

...

Function *FunctionAST::codegen() {
  // Chuyển quyền sở hữu prototype sang map FunctionProtos, nhưng giữ một
  // tham chiếu tới nó để dùng ở bên dưới.
  auto &P = *Proto;
  FunctionProtos[Proto->getName()] = std::move(Proto);
  Function *TheFunction = getFunction(P.getName());
  if (!TheFunction)
    return nullptr;
```

Để làm được điều này, ta bắt đầu bằng việc thêm một biến toàn cục mới, `FunctionProtos`, lưu prototype gần đây nhất của mỗi hàm. Ta cũng thêm một phương thức tiện ích, `getFunction()`, để thay thế các lời gọi `TheModule->getFunction()`. Phương thức tiện ích này tìm kiếm một khai báo hàm sẵn có trong `TheModule`, và nếu không tìm thấy thì chuyển sang sinh một khai báo mới từ FunctionProtos. Trong `CallExprAST::codegen()`, ta chỉ cần thay thế lời gọi `TheModule->getFunction()`. Trong `FunctionAST::codegen()`, ta cần cập nhật map FunctionProtos trước, rồi mới gọi `getFunction()`. Sau khi làm xong, ta luôn có thể lấy được khai báo hàm trong module hiện tại cho bất kỳ hàm nào đã được khai báo trước đó.

Ta cũng cần cập nhật HandleDefinition và HandleExtern:

```c++
static void HandleDefinition() {
  if (auto FnAST = ParseDefinition()) {
    if (auto *FnIR = FnAST->codegen()) {
      fprintf(stderr, "Read function definition:");
      FnIR->print(errs());
      fprintf(stderr, "\n");
      ExitOnErr(TheJIT->addModule(
          ThreadSafeModule(std::move(TheModule), std::move(TheContext))));
      InitializeModuleAndManagers();
    }
  } else {
    // Bỏ qua token để khôi phục lỗi.
     getNextToken();
  }
}

static void HandleExtern() {
  if (auto ProtoAST = ParseExtern()) {
    if (auto *FnIR = ProtoAST->codegen()) {
      fprintf(stderr, "Read extern: ");
      FnIR->print(errs());
      fprintf(stderr, "\n");
      FunctionProtos[ProtoAST->getName()] = std::move(ProtoAST);
    }
  } else {
    // Bỏ qua token để khôi phục lỗi.
    getNextToken();
  }
}
```

Trong HandleDefinition, ta thêm hai dòng để chuyển hàm vừa được định nghĩa vào JIT và mở một module mới. Trong HandleExtern, ta chỉ cần thêm một dòng để đưa prototype vào FunctionProtos.

:::{warning}
Kể từ LLVM-9, việc trùng lặp ký hiệu trong các module khác nhau không còn được phép. Điều đó có nghĩa là bạn không thể định nghĩa lại hàm trong Kaleidoscope như minh họa bên dưới. Hãy bỏ qua phần này.

Lý do là các API JIT OrcV2 mới hơn cố gắng bám sát các quy tắc của trình liên kết tĩnh và động, bao gồm cả việc từ chối các ký hiệu trùng lặp. Việc yêu cầu tên ký hiệu là duy nhất cho phép chúng tôi hỗ trợ biên dịch đồng thời các ký hiệu, dùng tên ký hiệu (duy nhất) làm khóa để theo dõi.
:::

Sau khi thực hiện các thay đổi này, hãy thử lại REPL của chúng ta (lần này tôi đã bỏ phần kết xuất các hàm ẩn danh, chắc bạn đã nắm được ý rồi :) :

```
ready> def foo(x) x + 1;
ready> foo(2);
Evaluated to 3.000000

ready> def foo(x) x + 2;
ready> foo(2);
Evaluated to 4.000000
```

Nó hoạt động rồi!

Ngay cả với đoạn mã đơn giản này, ta đã có được những khả năng mạnh mẽ đến bất ngờ - hãy xem này:

```
ready> extern sin(x);
Read extern:
declare double @sin(double)

ready> extern cos(x);
Read extern:
declare double @cos(double)

ready> sin(1.0);
Read top-level expression:
define double @2() {
entry:
  ret double 0x3FEAED548F090CEE
}

Evaluated to 0.841471

ready> def foo(x) sin(x)*sin(x) + cos(x)*cos(x);
Read function definition:
define double @foo(double %x) {
entry:
  %calltmp = call double @sin(double %x)
  %multmp = fmul double %calltmp, %calltmp
  %calltmp2 = call double @cos(double %x)
  %multmp4 = fmul double %calltmp2, %calltmp2
  %addtmp = fadd double %multmp, %multmp4
  ret double %addtmp
}

ready> foo(4.0);
Read top-level expression:
define double @3() {
entry:
  %calltmp = call double @foo(double 4.000000e+00)
  ret double %calltmp
}

Evaluated to 1.000000
```

Ồ, làm sao JIT biết về sin và cos? Câu trả lời đơn giản đến bất ngờ: KaleidoscopeJIT có một quy tắc phân giải ký hiệu (symbol resolution) rất rõ ràng mà nó dùng để tìm những ký hiệu không có trong một module cụ thể nào: Trước tiên, nó tìm trong tất cả các module đã được thêm vào JIT, từ mới nhất đến cũ nhất, để tìm định nghĩa mới nhất. Nếu không tìm thấy định nghĩa nào bên trong JIT, nó sẽ chuyển sang gọi "`dlsym("sin")`" trên chính tiến trình Kaleidoscope. Vì "`sin`" được định nghĩa bên trong không gian địa chỉ của JIT, nó chỉ đơn giản vá các lời gọi trong module để gọi trực tiếp phiên bản `sin` của libm. Nhưng trong một số trường hợp, điều này còn đi xa hơn nữa: vì sin và cos là tên của các hàm toán học tiêu chuẩn, bộ gập hằng số sẽ trực tiếp đánh giá các lời gọi hàm ra kết quả đúng khi chúng được gọi với hằng số, như trong "`sin(1.0)`" ở trên.

Trong tương lai, ta sẽ thấy việc tinh chỉnh quy tắc phân giải ký hiệu này có thể mở ra đủ loại tính năng hữu ích, từ bảo mật (giới hạn tập ký hiệu mà mã JIT có thể truy cập), đến sinh mã động dựa trên tên ký hiệu, và thậm chí cả biên dịch lười (lazy compilation).

Một lợi ích tức thì của quy tắc phân giải ký hiệu là giờ đây ta có thể mở rộng ngôn ngữ bằng cách viết mã C++ tùy ý để hiện thực các phép toán. Ví dụ, nếu ta thêm:

```c++
#ifdef _WIN32
#define DLLEXPORT __declspec(dllexport)
#else
#define DLLEXPORT
#endif

/// putchard - putchar nhận vào một double và trả về 0.
extern "C" DLLEXPORT double putchard(double X) {
  fputc((char)X, stderr);
  return 0;
}
```

Lưu ý rằng trên Windows, ta cần thực sự export các hàm vì bộ nạp ký hiệu động sẽ dùng `GetProcAddress` để tìm các ký hiệu.

Giờ đây ta có thể tạo ra đầu ra đơn giản trên console bằng những câu lệnh như: "`extern putchard(x); putchard(120);`", lệnh này in ra chữ 'x' thường trên console (120 là mã ASCII của 'x'). Mã tương tự có thể được dùng để hiện thực nhập/xuất tệp, nhập liệu từ console và nhiều khả năng khác trong Kaleidoscope.

Đến đây là kết thúc chương về JIT và bộ tối ưu hóa của hướng dẫn Kaleidoscope. Tại thời điểm này, ta có thể biên dịch một ngôn ngữ lập trình chưa đầy đủ Turing (non-Turing-complete), tối ưu hóa và biên dịch JIT nó theo cách do người dùng điều khiển. Tiếp theo, chúng ta sẽ tìm hiểu cách [mở rộng ngôn ngữ với các cấu trúc luồng điều khiển](LangImpl05.md), đồng thời giải quyết một số vấn đề thú vị của LLVM IR trên đường đi.

## Mã nguồn đầy đủ

Dưới đây là toàn bộ mã nguồn của ví dụ xuyên suốt, đã được bổ sung JIT và bộ tối ưu hóa của LLVM. Để build ví dụ này, hãy dùng:

```bash
# Biên dịch
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
# Chạy
./toy
```

Nếu bạn biên dịch trên Linux, hãy nhớ thêm cả tùy chọn "-rdynamic". Tùy chọn này đảm bảo các hàm bên ngoài được phân giải đúng lúc chạy.

Đây là mã nguồn:

```{literalinclude} ../../../examples/Kaleidoscope/Chapter4/toy.cpp
:language: c++
```

[Tiếp theo: Mở rộng ngôn ngữ: luồng điều khiển](LangImpl05.md)
