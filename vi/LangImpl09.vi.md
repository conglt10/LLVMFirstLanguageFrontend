# Kaleidoscope: Thêm thông tin gỡ lỗi

## Giới thiệu Chương 9

Chào mừng bạn đến với Chương 9 của loạt bài hướng dẫn "[Hiện thực một ngôn ngữ với LLVM](index.md)". Từ chương 1 đến chương 8, chúng ta đã xây dựng được một ngôn ngữ lập trình nhỏ nhưng khá ổn, có hàm và biến. Nhưng nếu có lỗi xảy ra thì sao, bạn sẽ gỡ lỗi chương trình của mình bằng cách nào?

Gỡ lỗi ở mức mã nguồn (source level debugging) sử dụng dữ liệu có định dạng giúp trình gỡ lỗi (debugger) ánh xạ từ mã nhị phân và trạng thái của máy ngược trở lại mã nguồn mà lập trình viên đã viết. Trong LLVM, chúng ta thường dùng một định dạng có tên là [DWARF](http://dwarfstd.org). DWARF là một cách mã hóa gọn nhẹ, biểu diễn các kiểu dữ liệu, vị trí trong mã nguồn và vị trí của biến.

Tóm tắt ngắn gọn, trong chương này chúng ta sẽ lần lượt đi qua những thứ cần thêm vào một ngôn ngữ lập trình để hỗ trợ thông tin gỡ lỗi (debug info), và cách chuyển chúng thành DWARF.

Lưu ý: Hiện tại chúng ta chưa thể gỡ lỗi thông qua JIT, vì vậy cần biên dịch chương trình thành một tệp thực thi nhỏ và độc lập. Để làm được điều này, chúng ta sẽ thực hiện một vài thay đổi đối với cách chạy ngôn ngữ và cách biên dịch chương trình. Điều đó có nghĩa là chúng ta sẽ có một tệp mã nguồn chứa một chương trình đơn giản viết bằng Kaleidoscope thay vì dùng JIT tương tác. Cách làm này có một hạn chế: mỗi lần chỉ được có một lệnh "cấp cao nhất" (top level), nhằm giảm số lượng thay đổi cần thiết.

Đây là chương trình mẫu mà chúng ta sẽ biên dịch:

```python
def fib(x)
  if x < 3 then
    1
  else
    fib(x-1)+fib(x-2);

fib(10)
```

## Tại sao đây là một bài toán khó?

Thông tin gỡ lỗi là một bài toán khó vì một vài lý do khác nhau, chủ yếu xoay quanh mã đã được tối ưu hóa. Thứ nhất, tối ưu hóa khiến việc giữ lại vị trí mã nguồn trở nên khó khăn hơn. Trong LLVM IR, chúng ta lưu vị trí mã nguồn gốc của mỗi lệnh ở mức IR ngay trên chính lệnh đó. Các pass tối ưu hóa nên giữ lại vị trí mã nguồn cho những lệnh mới được tạo ra, nhưng các lệnh bị gộp lại chỉ được giữ một vị trí duy nhất - điều này có thể gây ra hiện tượng nhảy lung tung khi chạy từng bước (stepping) qua chương trình đã tối ưu hóa. Thứ hai, tối ưu hóa có thể di chuyển biến theo những cách khiến biến bị loại bỏ hoàn toàn, dùng chung vùng nhớ với các biến khác, hoặc khó theo dõi. Trong phạm vi bài hướng dẫn này, chúng ta sẽ tránh tối ưu hóa (như bạn sẽ thấy trong một trong các nhóm bản vá tiếp theo).

## Chế độ biên dịch trước (Ahead-of-Time Compilation)

Để chỉ tập trung vào khía cạnh thêm thông tin gỡ lỗi cho một ngôn ngữ nguồn mà không phải bận tâm đến sự phức tạp của việc gỡ lỗi JIT, chúng ta sẽ thực hiện một vài thay đổi cho Kaleidoscope để hỗ trợ biên dịch IR do front end sinh ra thành một chương trình độc lập đơn giản mà bạn có thể chạy, gỡ lỗi và xem kết quả.

Trước tiên, chúng ta biến hàm ẩn danh chứa câu lệnh cấp cao nhất thành hàm "main":

```udiff
-    auto Proto = std::make_unique<PrototypeAST>("", std::vector<std::string>());
+    auto Proto = std::make_unique<PrototypeAST>("main", std::vector<std::string>());
```

chỉ với một thay đổi đơn giản là đặt tên cho nó.

Sau đó, chúng ta sẽ xóa đoạn mã liên quan đến dòng lệnh ở bất cứ đâu nó xuất hiện:

```udiff
@@ -1129,7 +1129,6 @@ static void HandleTopLevelExpression() {
 /// top ::= definition | external | expression | ';'
 static void MainLoop() {
   while (true) {
-    fprintf(stderr, "ready> ");
     switch (CurTok) {
     case tok_eof:
       return;
@@ -1184,7 +1183,6 @@ int main() {
   BinopPrecedence['*'] = 40; // highest.

   // Prime the first token.
-  fprintf(stderr, "ready> ");
   getNextToken();
```

Cuối cùng, chúng ta sẽ tắt tất cả các pass tối ưu hóa và JIT, để điều duy nhất xảy ra sau khi phân tích cú pháp và sinh mã xong là LLVM IR được xuất ra standard error:

```udiff
@@ -1108,17 +1108,8 @@ static void HandleExtern() {
 static void HandleTopLevelExpression() {
   // Evaluate a top-level expression into an anonymous function.
   if (auto FnAST = ParseTopLevelExpr()) {
-    if (auto *FnIR = FnAST->codegen()) {
-      // We're just doing this to make sure it executes.
-      TheExecutionEngine->finalizeObject();
-      // JIT the function, returning a function pointer.
-      void *FPtr = TheExecutionEngine->getPointerToFunction(FnIR);
-
-      // Cast it to the right type (takes no arguments, returns a double) so we
-      // can call it as a native function.
-      double (*FP)() = (double (*)())(intptr_t)FPtr;
-      // Ignore the return value for this.
-      (void)FP;
+    if (!FnAST->codegen()) {
+      fprintf(stderr, "Error generating code for top level expr");
     }
   } else {
     // Skip token for error recovery.
@@ -1439,11 +1459,11 @@ int main() {
   // target lays out data structures.
   TheModule->setDataLayout(TheExecutionEngine->getDataLayout());
   OurFPM.add(new DataLayoutPass());
+#if 0
   OurFPM.add(createBasicAliasAnalysisPass());
   // Promote allocas to registers.
   OurFPM.add(createPromoteMemoryToRegisterPass());
@@ -1218,7 +1210,7 @@ int main() {
   OurFPM.add(createGVNPass());
   // Simplify the control flow graph (deleting unreachable blocks, etc).
   OurFPM.add(createCFGSimplificationPass());
-
+  #endif
   OurFPM.doInitialization();

   // Set the global so the code gen can use this.
```

Nhóm thay đổi tương đối nhỏ này đưa chúng ta đến chỗ có thể biên dịch đoạn mã Kaleidoscope thành một chương trình thực thi bằng dòng lệnh sau:

```bash
Kaleidoscope-Ch9 < fib.ks | & clang -x ir -
```

lệnh này sẽ tạo ra tệp a.out/a.exe trong thư mục làm việc hiện tại.

## Đơn vị biên dịch (Compile Unit)

Vùng chứa cấp cao nhất cho một đoạn mã trong DWARF là compile unit (đơn vị biên dịch). Nó chứa dữ liệu về kiểu và hàm của một translation unit (đơn vị dịch) riêng lẻ (hiểu là: một tệp mã nguồn). Vì vậy, việc đầu tiên chúng ta cần làm là tạo một compile unit cho tệp fib.ks.

## Thiết lập việc sinh DWARF

Tương tự như lớp `IRBuilder`, chúng ta có lớp [DIBuilder](https://llvm.org/doxygen/classllvm_1_1DIBuilder.html) giúp xây dựng siêu dữ liệu gỡ lỗi (debug metadata) cho một tệp LLVM IR. Nó tương ứng 1:1 giống như `IRBuilder` với LLVM IR, nhưng có tên gọi dễ hiểu hơn. Để sử dụng nó, bạn cần quen thuộc với thuật ngữ DWARF hơn so với mức cần thiết khi làm việc với tên của `IRBuilder` và `Instruction`, nhưng nếu đọc qua tài liệu tổng quan về [Định dạng Metadata](../../SourceLevelDebugging.md) thì mọi thứ sẽ rõ ràng hơn một chút. Chúng ta sẽ dùng lớp này để xây dựng tất cả các mô tả ở mức IR. Hàm khởi tạo của nó nhận vào một module, nên chúng ta cần tạo nó ngay sau khi tạo module. Chúng ta để nó là một biến static toàn cục để sử dụng dễ dàng hơn một chút.

Tiếp theo, chúng ta sẽ tạo một vùng chứa nhỏ để lưu đệm (cache) một số dữ liệu hay dùng. Đầu tiên là compile unit, nhưng chúng ta cũng sẽ viết thêm một ít mã cho kiểu dữ liệu duy nhất của mình, vì không phải lo lắng về các biểu thức có nhiều kiểu khác nhau:

```c++
static std::unique_ptr<DIBuilder> DBuilder;

struct DebugInfo {
  DICompileUnit *TheCU;
  DIType *DblTy;

  DIType *getDoubleTy();
} KSDbgInfo;

DIType *DebugInfo::getDoubleTy() {
  if (DblTy)
    return DblTy;

  DblTy = DBuilder->createBasicType("double", 64, dwarf::DW_ATE_float);
  return DblTy;
}
```

Và sau đó, trong `main` khi chúng ta tạo module:

```c++
DBuilder = std::make_unique<DIBuilder>(*TheModule);

KSDbgInfo.TheCU = DBuilder->createCompileUnit(
    dwarf::DW_LANG_C, DBuilder->createFile("fib.ks", "."),
    "Kaleidoscope Compiler", false, "", 0);
```

Có một vài điều cần lưu ý ở đây. Thứ nhất, dù chúng ta đang tạo compile unit cho một ngôn ngữ tên là Kaleidoscope, chúng ta lại dùng hằng số ngôn ngữ của C. Lý do là trình gỡ lỗi không nhất thiết hiểu được quy ước gọi hàm (calling convention) hay ABI mặc định của một ngôn ngữ mà nó không nhận ra, và chúng ta tuân theo ABI của C trong quá trình sinh mã LLVM, nên đây là lựa chọn chính xác nhất. Điều này đảm bảo chúng ta thực sự có thể gọi hàm từ trình gỡ lỗi và để chúng thực thi. Thứ hai, bạn sẽ thấy "fib.ks" trong lời gọi `createCompileUnit`. Đây là giá trị mặc định được gán cứng, vì chúng ta dùng chuyển hướng shell (shell redirection) để đưa mã nguồn vào trình biên dịch Kaleidoscope. Trong một front end thông thường, bạn sẽ có tên tệp đầu vào và đặt nó vào vị trí đó.

Một điều cuối cùng trong quá trình sinh thông tin gỡ lỗi qua DIBuilder là chúng ta cần "hoàn tất" (finalize) thông tin gỡ lỗi. Lý do nằm ở API bên dưới của DIBuilder, nhưng hãy đảm bảo bạn thực hiện việc này ở gần cuối hàm main:

```c++
DBuilder->finalize();
```

trước khi xuất module ra.

## Hàm

Giờ đã có `Compile Unit` và vị trí mã nguồn, chúng ta có thể thêm định nghĩa hàm vào thông tin gỡ lỗi. Vì vậy, trong `FunctionAST::codegen()`, chúng ta thêm vài dòng mã để mô tả ngữ cảnh cho subprogram (chương trình con), trong trường hợp này là "File", và định nghĩa thực sự của chính hàm đó.

Đây là phần ngữ cảnh:

```c++
DIFile *Unit = DBuilder->createFile(KSDbgInfo.TheCU->getFilename(),
                                    KSDbgInfo.TheCU->getDirectory());
```

đoạn mã này cho chúng ta một DIFile bằng cách hỏi `Compile Unit` đã tạo ở trên về thư mục và tên tệp hiện tại. Sau đó, tạm thời chúng ta dùng vị trí mã nguồn bằng 0 (vì AST hiện chưa có thông tin vị trí mã nguồn) và xây dựng định nghĩa hàm:

```c++
DIScope *FContext = Unit;
unsigned LineNo = 0;
unsigned ScopeLine = 0;
DISubprogram *SP = DBuilder->createFunction(
    FContext, P.getName(), StringRef(), Unit, LineNo,
    CreateFunctionType(TheFunction->arg_size()),
    ScopeLine,
    DINode::FlagPrototyped,
    DISubprogram::SPFlagDefinition);
TheFunction->setSubprogram(SP);
```

và giờ chúng ta có một DISubprogram chứa tham chiếu đến toàn bộ siêu dữ liệu của hàm.

## Vị trí mã nguồn

Điều quan trọng nhất đối với thông tin gỡ lỗi là vị trí mã nguồn chính xác - nhờ đó mới có thể ánh xạ ngược lại mã nguồn của bạn. Tuy nhiên, chúng ta gặp một vấn đề: Kaleidoscope hoàn toàn không có thông tin vị trí mã nguồn trong lexer hay parser, nên chúng ta cần thêm nó vào.

```c++
struct SourceLocation {
  int Line;
  int Col;
};
static SourceLocation CurLoc;
static SourceLocation LexLoc = {1, 0};

static int advance() {
  int LastChar = getchar();

  if (LastChar == '\n' || LastChar == '\r') {
    LexLoc.Line++;
    LexLoc.Col = 0;
  } else
    LexLoc.Col++;
  return LastChar;
}
```

Trong đoạn mã này, chúng ta đã thêm chức năng theo dõi dòng và cột của "tệp nguồn". Khi phân tích từ vựng (lex) mỗi token, chúng ta đặt "vị trí từ vựng" (lexical location) hiện tại thành dòng và cột tương ứng với điểm bắt đầu của token. Chúng ta làm điều này bằng cách thay thế tất cả các lời gọi `getchar()` trước đây bằng hàm mới `advance()`, hàm này sẽ theo dõi thông tin vị trí. Sau đó, chúng ta thêm vị trí mã nguồn vào tất cả các lớp AST:

```c++
class ExprAST {
  SourceLocation Loc;

  public:
    ExprAST(SourceLocation Loc = CurLoc) : Loc(Loc) {}
    virtual ~ExprAST() {}
    virtual Value* codegen() = 0;
    int getLine() const { return Loc.Line; }
    int getCol() const { return Loc.Col; }
    virtual raw_ostream &dump(raw_ostream &out, int ind) {
      return out << ':' << getLine() << ':' << getCol() << '\n';
    }
```

và truyền nó xuống khi tạo một biểu thức mới:

```c++
LHS = std::make_unique<BinaryExprAST>(BinLoc, BinOp, std::move(LHS),
                                       std::move(RHS));
```

nhờ đó mỗi biểu thức và biến của chúng ta đều có vị trí.

Để đảm bảo mọi lệnh đều có thông tin vị trí mã nguồn phù hợp, chúng ta phải báo cho `Builder` mỗi khi chuyển sang một vị trí mã nguồn mới. Chúng ta dùng một hàm trợ giúp nhỏ cho việc này:

```c++
void DebugInfo::emitLocation(ExprAST *AST) {
  if (!AST)
    return Builder->SetCurrentDebugLocation(DebugLoc());
  DIScope *Scope;
  if (LexicalBlocks.empty())
    Scope = TheCU;
  else
    Scope = LexicalBlocks.back();
  Builder->SetCurrentDebugLocation(
      DILocation::get(Scope->getContext(), AST->getLine(), AST->getCol(), Scope));
}
```

Hàm này vừa cho `IRBuilder` chính biết chúng ta đang ở đâu, vừa cho biết chúng ta đang ở trong phạm vi (scope) nào. Phạm vi có thể ở mức compile unit, hoặc là khối từ vựng (lexical block) bao quanh gần nhất, chẳng hạn như hàm hiện tại. Để biểu diễn điều này, chúng ta tạo một ngăn xếp (stack) các phạm vi trong `DebugInfo`:

```c++
std::vector<DIScope *> LexicalBlocks;
```

và đẩy phạm vi (hàm) lên đỉnh ngăn xếp khi bắt đầu sinh mã cho mỗi hàm:

```c++
KSDbgInfo.LexicalBlocks.push_back(SP);
```

Ngoài ra, đừng quên lấy phạm vi ra khỏi ngăn xếp khi kết thúc việc sinh mã cho hàm:

```c++
// Lấy khối từ vựng của hàm ra khỏi ngăn xếp vì chúng ta đã thêm nó vào
// một cách vô điều kiện.
KSDbgInfo.LexicalBlocks.pop_back();
```

Sau đó, chúng ta đảm bảo sinh vị trí mỗi khi bắt đầu sinh mã cho một đối tượng AST mới:

```c++
KSDbgInfo.emitLocation(this);
```

## Biến

Giờ đã có hàm, chúng ta cần có khả năng in ra các biến trong phạm vi. Hãy thiết lập các đối số của hàm để có backtrace (vết ngăn xếp lời gọi) tử tế và xem các hàm được gọi như thế nào. Không cần nhiều mã lắm, và thông thường chúng ta xử lý việc này khi tạo các alloca cho đối số trong `FunctionAST::codegen`.

```c++
// Ghi lại các đối số của hàm vào map NamedValues.
NamedValues.clear();
unsigned ArgIdx = 0;
for (auto &Arg : TheFunction->args()) {
  // Tạo một alloca cho biến này.
  AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, Arg.getName());

  // Tạo một bộ mô tả gỡ lỗi cho biến.
  DILocalVariable *D = DBuilder->createParameterVariable(
      SP, Arg.getName(), ++ArgIdx, Unit, LineNo, KSDbgInfo.getDoubleTy(),
      true);

  DBuilder->insertDeclare(Alloca, D, DBuilder->createExpression(),
                          DILocation::get(SP->getContext(), LineNo, 0, SP),
                          Builder->GetInsertBlock());

  // Lưu giá trị ban đầu vào alloca.
  Builder->CreateStore(&Arg, Alloca);

  // Thêm đối số vào bảng ký hiệu biến.
  NamedValues[std::string(Arg.getName())] = Alloca;
}
```

Ở đây, trước tiên chúng ta tạo biến, cung cấp cho nó phạm vi (`SP`), tên, vị trí mã nguồn, kiểu, và vì nó là một đối số nên có cả chỉ số của đối số. Tiếp theo, chúng ta tạo một bản ghi `#dbg_declare` để chỉ ra ở mức IR rằng có một biến nằm trong một alloca (đồng thời cung cấp vị trí bắt đầu cho biến), và đặt vị trí mã nguồn cho phần khai báo tại điểm bắt đầu của phạm vi.

Một điều thú vị cần lưu ý lúc này là các trình gỡ lỗi khác nhau có những giả định dựa trên cách mã và thông tin gỡ lỗi được sinh ra cho chúng trong quá khứ. Trong trường hợp này, chúng ta cần dùng một mẹo nhỏ để tránh sinh thông tin dòng cho phần mở đầu của hàm (function prologue), để trình gỡ lỗi biết cần bỏ qua các lệnh đó khi đặt breakpoint (điểm dừng). Vì vậy, trong `FunctionAST::CodeGen`, chúng ta thêm vài dòng nữa:

```c++
// Bỏ đặt vị trí khi sinh phần prologue (các lệnh đầu tiên không có vị trí
// trong một hàm được coi là một phần của prologue, và trình gỡ lỗi sẽ chạy
// qua chúng khi dừng tại một hàm)
KSDbgInfo.emitLocation(nullptr);
```

và sau đó sinh một vị trí mới khi thực sự bắt đầu sinh mã cho thân hàm:

```c++
KSDbgInfo.emitLocation(Body.get());
```

Với những điều này, chúng ta đã có đủ thông tin gỡ lỗi để đặt breakpoint trong các hàm, in ra các biến đối số và gọi hàm. Không tệ chút nào chỉ với vài dòng mã đơn giản!

## Mã nguồn đầy đủ

Dưới đây là mã nguồn đầy đủ cho ví dụ xuyên suốt của chúng ta, đã được bổ sung thông tin gỡ lỗi. Để build ví dụ này, hãy dùng:

```bash
# Biên dịch
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
# Chạy
./toy
```

Đây là mã nguồn:

```{literalinclude} ../../../examples/Kaleidoscope/Chapter9/toy.cpp
:language: c++
```

[Tiếp theo: Kết luận và một số mẹo hữu ích khác về LLVM](LangImpl10.md)
