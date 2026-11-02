# Kaleidoscope: Mở rộng ngôn ngữ: Biến khả biến

## Giới thiệu Chương 7

Chào mừng bạn đến với Chương 7 của hướng dẫn "[Hiện thực một ngôn ngữ với LLVM](index.md)". Từ chương 1 đến chương 6, chúng ta đã xây dựng được một [ngôn ngữ lập trình hàm](http://en.wikipedia.org/wiki/Functional_programming) khá tươm tất, dù còn đơn giản. Trong hành trình đó, chúng ta đã học một số kỹ thuật phân tích cú pháp (parsing), cách xây dựng và biểu diễn AST, cách sinh LLVM IR, cũng như cách tối ưu hóa mã kết quả và biên dịch JIT cho nó.

Dù Kaleidoscope thú vị ở chỗ là một ngôn ngữ hàm, chính việc nó là ngôn ngữ hàm lại khiến việc sinh LLVM IR cho nó trở nên "quá dễ". Cụ thể, một ngôn ngữ hàm giúp ta rất dễ xây dựng LLVM IR trực tiếp ở [dạng SSA](http://en.wikipedia.org/wiki/Static_single_assignment_form) (Static Single Assignment). Vì LLVM yêu cầu mã đầu vào phải ở dạng SSA, đây là một tính chất rất tốt, và những người mới thường không rõ làm thế nào để sinh mã cho một ngôn ngữ mệnh lệnh có biến khả biến (mutable variable).

Tóm tắt ngắn gọn (và đáng mừng) của chương này là: front-end của bạn không cần phải tự xây dựng dạng SSA. LLVM cung cấp sẵn cơ chế hỗ trợ được tinh chỉnh kỹ lưỡng và kiểm thử cẩn thận cho việc này, dù cách nó hoạt động có phần bất ngờ với một số người.

## Tại sao đây là một bài toán khó?

Để hiểu vì sao biến khả biến gây phức tạp cho việc xây dựng SSA, hãy xem ví dụ C cực kỳ đơn giản sau:

```c
int G, H;
int test(_Bool Condition) {
  int X;
  if (Condition)
    X = G;
  else
    X = H;
  return X;
}
```

Trong trường hợp này, ta có biến "X" mà giá trị phụ thuộc vào đường thực thi trong chương trình. Vì X có thể mang hai giá trị khác nhau trước lệnh return, một nút PHI được chèn vào để hợp nhất hai giá trị này. LLVM IR mà ta muốn cho ví dụ này trông như sau:

```llvm
@G = weak global i32 0   ; kiểu của @G là i32*
@H = weak global i32 0   ; kiểu của @H là i32*

define i32 @test(i1 %Condition) {
entry:
  br i1 %Condition, label %cond_true, label %cond_false

cond_true:
  %X.0 = load i32, i32* @G
  br label %cond_next

cond_false:
  %X.1 = load i32, i32* @H
  br label %cond_next

cond_next:
  %X.2 = phi i32 [ %X.1, %cond_false ], [ %X.0, %cond_true ]
  ret i32 %X.2
}
```

Trong ví dụ này, các lệnh load từ biến toàn cục G và H được thể hiện tường minh trong LLVM IR, và chúng nằm trong các nhánh then/else của câu lệnh if (cond_true/cond_false). Để hợp nhất các giá trị đi vào, nút phi X.2 trong khối cond_next chọn giá trị đúng dựa trên nơi luồng điều khiển đi tới: nếu luồng điều khiển đến từ khối cond_false, X.2 nhận giá trị của X.1; ngược lại, nếu luồng điều khiển đến từ cond_true, nó nhận giá trị của X.0. Mục đích của chương này không phải là giải thích chi tiết về dạng SSA. Để biết thêm thông tin, hãy xem một trong nhiều [tài liệu tham khảo trực tuyến](http://en.wikipedia.org/wiki/Static_single_assignment_form).

Câu hỏi đặt ra cho bài viết này là "ai sẽ đặt các nút phi khi hạ (lowering) các phép gán cho biến khả biến?". Vấn đề ở đây là LLVM *yêu cầu* IR của nó phải ở dạng SSA: không có chế độ "phi SSA" nào cả. Tuy nhiên, việc xây dựng SSA đòi hỏi các thuật toán và cấu trúc dữ liệu không hề đơn giản, nên sẽ rất bất tiện và lãng phí nếu mỗi front-end đều phải tự tái hiện logic này.

## Bộ nhớ trong LLVM

"Mẹo" ở đây là: tuy LLVM yêu cầu mọi giá trị thanh ghi phải ở dạng SSA, nó không yêu cầu (và cũng không cho phép) các đối tượng bộ nhớ ở dạng SSA. Trong ví dụ trên, hãy để ý rằng các lệnh load từ G và H là truy cập trực tiếp vào G và H: chúng không bị đổi tên hay đánh phiên bản. Điều này khác với một số hệ thống trình biên dịch khác, vốn cố gắng đánh phiên bản cho các đối tượng bộ nhớ. Trong LLVM, thay vì mã hóa phân tích luồng dữ liệu của bộ nhớ vào LLVM IR, việc này được xử lý bằng các [Analysis Pass](../../WritingAnLLVMPass.md) (pass phân tích) được tính toán khi cần.

Với điều này trong đầu, ý tưởng tổng quát là ta muốn tạo một biến trên ngăn xếp (stack) — vốn nằm trong bộ nhớ, vì nó ở trên stack — cho mỗi đối tượng khả biến trong một hàm. Để tận dụng mẹo này, ta cần nói về cách LLVM biểu diễn biến trên stack.

Trong LLVM, mọi truy cập bộ nhớ đều tường minh thông qua các lệnh load/store, và nó được thiết kế cẩn thận để không có (hay cần) toán tử "lấy địa chỉ" (address-of). Hãy để ý rằng kiểu của các biến toàn cục @G/@H thực ra là `i32*` dù biến được định nghĩa là "i32". Điều này có nghĩa là @G định nghĩa một *vùng nhớ* cho một i32 trong vùng dữ liệu toàn cục, nhưng *tên* của nó thực ra tham chiếu đến địa chỉ của vùng nhớ đó. Biến trên stack hoạt động theo cách tương tự, chỉ khác là thay vì được khai báo bằng định nghĩa biến toàn cục, chúng được khai báo bằng [lệnh alloca của LLVM](../../LangRef.md#alloca-instruction):

```llvm
define i32 @example() {
entry:
  %X = alloca i32           ; kiểu của %X là i32*.
  ...
  %tmp = load i32, i32* %X  ; nạp giá trị %X từ stack.
  %tmp2 = add i32 %tmp, 1   ; tăng nó lên
  store i32 %tmp2, i32* %X  ; lưu nó trở lại
  ...
```

Đoạn mã này cho thấy một ví dụ về cách khai báo và thao tác với một biến trên stack trong LLVM IR. Bộ nhớ stack được cấp phát bằng lệnh alloca hoàn toàn tổng quát: bạn có thể truyền địa chỉ của ô stack (stack slot) vào hàm, lưu nó vào các biến khác, v.v. Với ví dụ ở trên, ta có thể viết lại bằng kỹ thuật alloca để tránh dùng nút PHI:

```llvm
@G = weak global i32 0   ; kiểu của @G là i32*
@H = weak global i32 0   ; kiểu của @H là i32*

define i32 @test(i1 %Condition) {
entry:
  %X = alloca i32           ; kiểu của %X là i32*.
  br i1 %Condition, label %cond_true, label %cond_false

cond_true:
  %X.0 = load i32, i32* @G
  store i32 %X.0, i32* %X   ; Cập nhật X
  br label %cond_next

cond_false:
  %X.1 = load i32, i32* @H
  store i32 %X.1, i32* %X   ; Cập nhật X
  br label %cond_next

cond_next:
  %X.2 = load i32, i32* %X  ; Đọc X
  ret i32 %X.2
}
```

Như vậy, ta đã tìm ra một cách xử lý biến khả biến bất kỳ mà hoàn toàn không cần tạo nút Phi:

1. Mỗi biến khả biến trở thành một vùng cấp phát trên stack.
2. Mỗi lần đọc biến trở thành một lệnh load từ stack.
3. Mỗi lần cập nhật biến trở thành một lệnh store vào stack.
4. Việc lấy địa chỉ của một biến chỉ đơn giản là dùng trực tiếp địa chỉ trên stack.

Dù giải pháp này đã giải quyết được vấn đề trước mắt, nó lại sinh ra một vấn đề khác: giờ đây ta dường như đã tạo ra rất nhiều lưu lượng truy cập stack cho những thao tác rất đơn giản và phổ biến — một vấn đề lớn về hiệu năng. May mắn thay, bộ tối ưu hóa của LLVM có một pass tối ưu hóa được tinh chỉnh kỹ lưỡng tên là "mem2reg" để xử lý trường hợp này: nó nâng (promote) các alloca như thế lên thành thanh ghi SSA, đồng thời chèn nút Phi khi thích hợp. Ví dụ, nếu chạy ví dụ trên qua pass này, bạn sẽ nhận được:

```bash
$ llvm-as < example.ll | opt -passes=mem2reg | llvm-dis
@G = weak global i32 0
@H = weak global i32 0

define i32 @test(i1 %Condition) {
entry:
  br i1 %Condition, label %cond_true, label %cond_false

cond_true:
  %X.0 = load i32, i32* @G
  br label %cond_next

cond_false:
  %X.1 = load i32, i32* @H
  br label %cond_next

cond_next:
  %X.01 = phi i32 [ %X.1, %cond_false ], [ %X.0, %cond_true ]
  ret i32 %X.01
}
```

Pass mem2reg hiện thực thuật toán chuẩn "iterated dominance frontier" (biên trội lặp) để xây dựng dạng SSA và có một số tối ưu hóa giúp tăng tốc các trường hợp suy biến (rất phổ biến). Pass tối ưu hóa mem2reg chính là câu trả lời cho việc xử lý biến khả biến, và chúng tôi đặc biệt khuyên bạn nên dựa vào nó. Lưu ý rằng mem2reg chỉ hoạt động với biến trong một số điều kiện nhất định:

1. mem2reg hoạt động dựa trên alloca: nó tìm các alloca và nếu xử lý được thì sẽ nâng chúng lên thanh ghi. Nó không áp dụng cho biến toàn cục hay các vùng cấp phát trên heap.
2. mem2reg chỉ tìm các lệnh alloca trong khối entry của hàm. Việc nằm trong khối entry đảm bảo rằng alloca chỉ được thực thi một lần, giúp việc phân tích trở nên đơn giản hơn.
3. mem2reg chỉ nâng các alloca mà mọi cách dùng đều là load và store trực tiếp. Nếu địa chỉ của đối tượng trên stack được truyền vào một hàm, hoặc nếu có bất kỳ phép toán con trỏ "kỳ quặc" nào, alloca đó sẽ không được nâng.
4. mem2reg chỉ hoạt động với alloca của các giá trị [first class](../../LangRef.md#first-class-types) (như con trỏ, số vô hướng và vector), và chỉ khi kích thước mảng của vùng cấp phát là 1 (hoặc bị bỏ trống trong file .ll). mem2reg không có khả năng nâng struct hay mảng lên thanh ghi. Lưu ý rằng pass "sroa" mạnh hơn và có thể nâng struct, "union" và mảng trong nhiều trường hợp.

Tất cả các tính chất này đều dễ thỏa mãn với hầu hết các ngôn ngữ mệnh lệnh, và chúng ta sẽ minh họa điều đó bên dưới với Kaleidoscope. Câu hỏi cuối cùng mà có thể bạn đang tự hỏi là: tôi có nên bận tâm đến chuyện vớ vẩn này cho front-end của mình không? Chẳng phải sẽ tốt hơn nếu tôi tự xây dựng SSA trực tiếp, tránh dùng pass tối ưu hóa mem2reg sao? Nói ngắn gọn, chúng tôi đặc biệt khuyên bạn dùng kỹ thuật này để xây dựng dạng SSA, trừ khi có lý do cực kỳ chính đáng để không làm vậy. Sử dụng kỹ thuật này:

- Đã được chứng minh và kiểm thử kỹ: clang dùng kỹ thuật này cho các biến cục bộ khả biến. Như vậy, những client phổ biến nhất của LLVM đang dùng nó để xử lý phần lớn các biến của họ. Bạn có thể yên tâm rằng lỗi sẽ được phát hiện nhanh và sửa sớm.
- Cực kỳ nhanh: mem2reg có một số trường hợp đặc biệt giúp nó nhanh trong các trường hợp phổ biến mà vẫn hoàn toàn tổng quát. Ví dụ, nó có các đường xử lý nhanh (fast-path) cho biến chỉ được dùng trong một khối duy nhất, biến chỉ có một điểm gán, các heuristic tốt để tránh chèn những nút phi không cần thiết, v.v.
- Cần thiết cho việc sinh thông tin gỡ lỗi: [Thông tin gỡ lỗi trong LLVM](../../SourceLevelDebugging.md) dựa vào việc địa chỉ của biến được phơi ra để có thể gắn thông tin gỡ lỗi vào đó. Kỹ thuật này ăn khớp rất tự nhiên với cách tổ chức thông tin gỡ lỗi như vậy.

Chí ít thì kỹ thuật này cũng giúp front-end của bạn dễ dàng chạy được hơn nhiều, và rất đơn giản để hiện thực. Giờ hãy mở rộng Kaleidoscope với biến khả biến nào!

## Biến khả biến trong Kaleidoscope

Giờ ta đã biết loại vấn đề mình muốn giải quyết, hãy xem nó trông như thế nào trong bối cảnh ngôn ngữ Kaleidoscope nhỏ bé của chúng ta. Chúng ta sẽ thêm hai tính năng:

1. Khả năng thay đổi giá trị biến bằng toán tử '='.
2. Khả năng định nghĩa biến mới.

Dù mục đầu tiên mới thực sự là trọng tâm, hiện tại ta chỉ có biến cho các đối số đầu vào và cho biến cảm ứng (induction variable), và việc định nghĩa lại chúng thì cũng chỉ giúp được đến thế thôi :). Ngoài ra, khả năng định nghĩa biến mới là một điều hữu ích bất kể bạn có thay đổi giá trị của chúng hay không. Dưới đây là một ví dụ minh họa cho thấy cách ta có thể dùng các tính năng này:

```
# Định nghĩa ':' để nối tiếp: là một toán tử có độ ưu tiên thấp, bỏ qua các toán hạng
# và chỉ trả về RHS.
def binary : 1 (x y) y;

# Hàm fib đệ quy, trước đây ta đã làm được.
def fib(x)
  if (x < 3) then
    1
  else
    fib(x-1)+fib(x-2);

# Hàm fib dạng lặp.
def fibi(x)
  var a = 1, b = 1, c in
  (for i = 3, i < x in
     c = a + b :
     a = b :
     b = c) :
  b;

# Gọi hàm.
fibi(10);
```

Để có thể thay đổi giá trị biến, ta phải sửa các biến hiện có để dùng "mẹo alloca". Khi đã có điều đó, ta sẽ thêm toán tử mới, rồi mở rộng Kaleidoscope để hỗ trợ định nghĩa biến mới.

## Điều chỉnh các biến hiện có để hỗ trợ thay đổi giá trị

Bảng ký hiệu (symbol table) trong Kaleidoscope được quản lý tại thời điểm sinh mã bởi map '`NamedValues`'. Map này hiện lưu `Value*` của LLVM chứa giá trị double cho biến được đặt tên. Để hỗ trợ thay đổi giá trị, ta cần sửa đổi nó một chút để `NamedValues` lưu *vị trí bộ nhớ* của biến tương ứng. Lưu ý rằng thay đổi này là một lần tái cấu trúc (refactoring): nó thay đổi cấu trúc mã nhưng (tự nó) không thay đổi hành vi của trình biên dịch. Tất cả các thay đổi này đều nằm gọn trong bộ sinh mã của Kaleidoscope.

Ở giai đoạn phát triển hiện tại, Kaleidoscope chỉ hỗ trợ biến cho hai thứ: đối số đầu vào của hàm và biến cảm ứng của vòng lặp 'for'. Để nhất quán, ta sẽ cho phép thay đổi giá trị của những biến này cùng với các biến do người dùng định nghĩa khác. Điều này có nghĩa là cả hai loại đều cần có vị trí bộ nhớ.

Để bắt đầu chuyển đổi Kaleidoscope, ta sẽ thay đổi map `NamedValues` để nó ánh xạ tới `AllocaInst*` thay vì `Value*`. Khi làm vậy, trình biên dịch C++ sẽ cho ta biết những phần nào của mã cần cập nhật:

```c++
static std::map<std::string, AllocaInst*> NamedValues;
```

Ngoài ra, vì ta sẽ cần tạo các alloca này, ta sẽ dùng một hàm trợ giúp đảm bảo rằng các alloca được tạo trong khối entry của hàm:

```c++
/// CreateEntryBlockAlloca - Tạo một lệnh alloca trong khối entry của
/// hàm. Hàm này được dùng cho biến khả biến, v.v.
static AllocaInst *CreateEntryBlockAlloca(Function *TheFunction,
                                          StringRef VarName) {
  IRBuilder<> TmpB(&TheFunction->getEntryBlock(),
                 TheFunction->getEntryBlock().begin());
  return TmpB.CreateAlloca(Type::getDoubleTy(*TheContext), nullptr,
                           VarName);
}
```

Đoạn mã trông hơi lạ này tạo một đối tượng IRBuilder trỏ vào lệnh đầu tiên (.begin()) của khối entry. Sau đó nó tạo một alloca với tên mong muốn và trả về alloca đó. Vì mọi giá trị trong Kaleidoscope đều là double, không cần truyền vào kiểu dữ liệu.

Với điều này, thay đổi chức năng đầu tiên mà ta muốn thực hiện là ở phần tham chiếu biến. Trong cơ chế mới, biến nằm trên stack, nên mã sinh ra cho một tham chiếu tới biến thực chất cần tạo một lệnh load từ ô stack:

```c++
Value *VariableExprAST::codegen() {
  // Tìm biến này trong hàm.
  AllocaInst *A = NamedValues[Name];
  if (!A)
    return LogErrorV("Unknown variable name");

  // Nạp giá trị.
  return Builder->CreateLoad(A->getAllocatedType(), A, Name.c_str());
}
```

Như bạn thấy, việc này khá đơn giản. Giờ ta cần cập nhật những chỗ định nghĩa biến để thiết lập alloca. Ta sẽ bắt đầu với `ForExprAST::codegen()` (xem [toàn bộ mã nguồn](#full-code-listing) để có mã đầy đủ):

```c++
Function *TheFunction = Builder->GetInsertBlock()->getParent();

// Tạo một alloca cho biến trong khối entry.
AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, VarName);

// Sinh mã cho giá trị bắt đầu trước, khi 'biến' chưa nằm trong phạm vi.
Value *StartVal = Start->codegen();
if (!StartVal)
  return nullptr;

// Lưu giá trị vào alloca.
Builder->CreateStore(StartVal, Alloca);
...

// Tính điều kiện kết thúc.
Value *EndCond = End->codegen();
if (!EndCond)
  return nullptr;

// Nạp lại, tăng giá trị, và lưu lại vào alloca. Điều này xử lý trường hợp
// thân vòng lặp thay đổi giá trị của biến.
Value *CurVar = Builder->CreateLoad(Alloca->getAllocatedType(), Alloca,
                                    VarName.c_str());
Value *NextVar = Builder->CreateFAdd(CurVar, StepVal, "nextvar");
Builder->CreateStore(NextVar, Alloca);
...
```

Đoạn mã này gần như giống hệt mã [trước khi ta cho phép biến khả biến](LangImpl05.md#code-generation-for-the-for-loop). Khác biệt lớn là ta không còn phải tạo nút PHI nữa, và ta dùng load/store để truy cập biến khi cần.

Để hỗ trợ biến đối số khả biến, ta cũng cần tạo alloca cho chúng. Mã cho việc này cũng khá đơn giản:

```c++
Function *FunctionAST::codegen() {
  ...
  Builder->SetInsertPoint(BB);

  // Ghi nhận các đối số của hàm vào map NamedValues.
  NamedValues.clear();
  for (auto &Arg : TheFunction->args()) {
    // Tạo một alloca cho biến này.
    AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, Arg.getName());

    // Lưu giá trị ban đầu vào alloca.
    Builder->CreateStore(&Arg, Alloca);

    // Thêm các đối số vào bảng ký hiệu biến.
    NamedValues[std::string(Arg.getName())] = Alloca;
  }

  if (Value *RetVal = Body->codegen()) {
    ...
```

Với mỗi đối số, ta tạo một alloca, lưu giá trị đầu vào của hàm vào alloca đó, và đăng ký alloca làm vị trí bộ nhớ cho đối số. Phương thức này được `FunctionAST::codegen()` gọi ngay sau khi nó thiết lập khối entry cho hàm.

Mảnh ghép còn thiếu cuối cùng là thêm pass mem2reg, giúp ta một lần nữa có được mã sinh ra chất lượng tốt:

```c++
// Nâng các alloca lên thanh ghi.
TheFPM->addPass(PromotePass());
// Thực hiện các tối ưu hóa "peephole" đơn giản và tối ưu thao tác bit.
TheFPM->addPass(InstCombinePass());
// Tái kết hợp các biểu thức.
TheFPM->addPass(ReassociatePass());
...
```

Thật thú vị khi xem mã trông như thế nào trước và sau khi tối ưu hóa mem2reg chạy. Ví dụ, đây là mã trước/sau cho hàm fib đệ quy của chúng ta. Trước khi tối ưu hóa:

```llvm
define double @fib(double %x) {
entry:
  %x1 = alloca double
  store double %x, double* %x1
  %x2 = load double, double* %x1
  %cmptmp = fcmp ult double %x2, 3.000000e+00
  %booltmp = uitofp i1 %cmptmp to double
  %ifcond = fcmp one double %booltmp, 0.000000e+00
  br i1 %ifcond, label %then, label %else

then:       ; preds = %entry
  br label %ifcont

else:       ; preds = %entry
  %x3 = load double, double* %x1
  %subtmp = fsub double %x3, 1.000000e+00
  %calltmp = call double @fib(double %subtmp)
  %x4 = load double, double* %x1
  %subtmp5 = fsub double %x4, 2.000000e+00
  %calltmp6 = call double @fib(double %subtmp5)
  %addtmp = fadd double %calltmp, %calltmp6
  br label %ifcont

ifcont:     ; preds = %else, %then
  %iftmp = phi double [ 1.000000e+00, %then ], [ %addtmp, %else ]
  ret double %iftmp
}
```

Ở đây chỉ có một biến (x, đối số đầu vào) nhưng bạn vẫn có thể thấy chiến lược sinh mã cực kỳ ngây ngô mà ta đang dùng. Trong khối entry, một alloca được tạo ra, và giá trị đầu vào ban đầu được lưu vào đó. Mỗi tham chiếu tới biến đều nạp lại từ stack. Ngoài ra, lưu ý rằng ta không sửa biểu thức if/then/else, nên nó vẫn chèn nút PHI. Dù ta có thể tạo alloca cho nó, thực ra tạo nút PHI lại dễ hơn, nên ta vẫn cứ tạo PHI.

Đây là mã sau khi pass mem2reg chạy:

```llvm
define double @fib(double %x) {
entry:
  %cmptmp = fcmp ult double %x, 3.000000e+00
  %booltmp = uitofp i1 %cmptmp to double
  %ifcond = fcmp one double %booltmp, 0.000000e+00
  br i1 %ifcond, label %then, label %else

then:
  br label %ifcont

else:
  %subtmp = fsub double %x, 1.000000e+00
  %calltmp = call double @fib(double %subtmp)
  %subtmp5 = fsub double %x, 2.000000e+00
  %calltmp6 = call double @fib(double %subtmp5)
  %addtmp = fadd double %calltmp, %calltmp6
  br label %ifcont

ifcont:     ; preds = %else, %then
  %iftmp = phi double [ 1.000000e+00, %then ], [ %addtmp, %else ]
  ret double %iftmp
}
```

Đây là một trường hợp tầm thường đối với mem2reg, vì không có lần định nghĩa lại nào của biến. Mục đích của việc trình bày điều này là để bạn bớt lo lắng về việc chèn vào những đoạn mã kém hiệu quả lộ liễu như vậy :).

Sau khi phần còn lại của các bộ tối ưu hóa chạy, ta nhận được:

```llvm
define double @fib(double %x) {
entry:
  %cmptmp = fcmp ult double %x, 3.000000e+00
  %booltmp = uitofp i1 %cmptmp to double
  %ifcond = fcmp ueq double %booltmp, 0.000000e+00
  br i1 %ifcond, label %else, label %ifcont

else:
  %subtmp = fsub double %x, 1.000000e+00
  %calltmp = call double @fib(double %subtmp)
  %subtmp5 = fsub double %x, 2.000000e+00
  %calltmp6 = call double @fib(double %subtmp5)
  %addtmp = fadd double %calltmp, %calltmp6
  ret double %addtmp

ifcont:
  ret double 1.000000e+00
}
```

Ở đây ta thấy pass simplifycfg đã quyết định sao chép lệnh return vào cuối khối 'else'. Điều này cho phép nó loại bỏ một số lệnh rẽ nhánh và nút PHI.

Giờ khi mọi tham chiếu trong bảng ký hiệu đã được cập nhật để dùng biến trên stack, ta sẽ thêm toán tử gán.

## Toán tử gán mới

Với khung sườn hiện tại, việc thêm một toán tử gán mới thực sự rất đơn giản. Ta sẽ phân tích cú pháp nó giống như mọi toán tử hai ngôi khác, nhưng xử lý nó ở bên trong (thay vì cho phép người dùng định nghĩa). Bước đầu tiên là đặt độ ưu tiên:

```c++
int main() {
  // Cài đặt các toán tử hai ngôi chuẩn.
  // 1 là độ ưu tiên thấp nhất.
  BinopPrecedence['='] = 2;
  BinopPrecedence['<'] = 10;
  BinopPrecedence['+'] = 20;
  BinopPrecedence['-'] = 20;
```

Giờ parser đã biết độ ưu tiên của toán tử hai ngôi, nó sẽ lo toàn bộ việc phân tích cú pháp và sinh AST. Ta chỉ cần hiện thực codegen cho toán tử gán. Mã trông như sau:

```c++
Value *BinaryExprAST::codegen() {
  // Xử lý đặc biệt '=' vì ta không muốn sinh mã cho LHS như một biểu thức.
  if (Op == '=') {
    // Ở đây giả định ta build mà không có RTTI vì mặc định LLVM được build
    // như vậy. Nếu bạn build LLVM với RTTI, có thể đổi thành dynamic_cast
    // để kiểm tra lỗi tự động.
    VariableExprAST *LHSE = static_cast<VariableExprAST*>(LHS.get());
    if (!LHSE)
      return LogErrorV("destination of '=' must be a variable");
```

Không giống các toán tử hai ngôi còn lại, toán tử gán của ta không tuân theo mô hình "sinh mã LHS, sinh mã RHS, thực hiện phép tính". Vì vậy, nó được xử lý như một trường hợp đặc biệt trước khi các toán tử hai ngôi khác được xử lý. Một điểm lạ khác là nó yêu cầu LHS phải là một biến. Viết "(x+1) = expr" là không hợp lệ — chỉ những thứ như "x = expr" mới được cho phép.

```c++
  // Sinh mã cho RHS.
  Value *Val = RHS->codegen();
  if (!Val)
    return nullptr;

  // Tra cứu tên.
  AllocaInst *Alloca = NamedValues[LHSE->getName()];
  if (!Alloca)
    return LogErrorV("Unknown variable name");

  Builder->CreateStore(Val, Alloca);
  return Val;
}
...
```

Khi đã có biến, việc sinh mã cho phép gán rất đơn giản: ta sinh mã cho RHS của phép gán, tạo một lệnh store, và trả về giá trị đã tính. Việc trả về một giá trị cho phép gán theo chuỗi như "X = (Y = Z)".

Giờ khi đã có toán tử gán, ta có thể thay đổi giá trị của biến vòng lặp và đối số. Ví dụ, giờ ta có thể chạy mã như sau:

```
# Hàm in ra một số double.
extern printd(x);

# Định nghĩa ':' để nối tiếp: là một toán tử có độ ưu tiên thấp, bỏ qua các toán hạng
# và chỉ trả về RHS.
def binary : 1 (x y) y;

def test(x)
  printd(x) :
  x = 4 :
  printd(x);

test(123);
```

Khi chạy, ví dụ này in ra "123" rồi "4", cho thấy ta đã thực sự thay đổi được giá trị! Được rồi, giờ ta đã chính thức đạt được mục tiêu: để chạy được điều này, trong trường hợp tổng quát cần phải xây dựng SSA. Tuy nhiên, để thực sự hữu ích, ta muốn có khả năng định nghĩa biến cục bộ của riêng mình — hãy thêm tính năng đó tiếp theo!

## Biến cục bộ do người dùng định nghĩa

Việc thêm var/in cũng giống như mọi phần mở rộng khác mà ta đã làm cho Kaleidoscope: ta mở rộng lexer, parser, AST và bộ sinh mã. Bước đầu tiên để thêm cấu trúc 'var/in' mới là mở rộng lexer. Như trước đây, việc này khá đơn giản, mã trông như sau:

```c++
enum Token {
  ...
  // định nghĩa var
  tok_var = -13
...
}
...
static int gettok() {
...
    if (IdentifierStr == "in")
      return tok_in;
    if (IdentifierStr == "binary")
      return tok_binary;
    if (IdentifierStr == "unary")
      return tok_unary;
    if (IdentifierStr == "var")
      return tok_var;
    return tok_identifier;
...
```

Bước tiếp theo là định nghĩa nút AST mà ta sẽ tạo. Với var/in, nó trông như sau:

```c++
/// VarExprAST - Lớp biểu thức cho var/in
class VarExprAST : public ExprAST {
  std::vector<std::pair<std::string, std::unique_ptr<ExprAST>>> VarNames;
  std::unique_ptr<ExprAST> Body;

public:
  VarExprAST(std::vector<std::pair<std::string, std::unique_ptr<ExprAST>>> VarNames,
             std::unique_ptr<ExprAST> Body)
    : VarNames(std::move(VarNames)), Body(std::move(Body)) {}

  Value *codegen() override;
};
```

var/in cho phép định nghĩa một danh sách tên cùng lúc, và mỗi tên có thể tùy chọn có một giá trị khởi tạo. Vì vậy, ta lưu thông tin này trong vector VarNames. Ngoài ra, var/in có một phần thân, và phần thân này được phép truy cập các biến được định nghĩa bởi var/in.

Với điều này, ta có thể định nghĩa các phần của parser. Việc đầu tiên là thêm nó làm một biểu thức sơ cấp (primary expression):

```c++
/// primary
///   ::= identifierexpr
///   ::= numberexpr
///   ::= parenexpr
///   ::= ifexpr
///   ::= forexpr
///   ::= varexpr
static std::unique_ptr<ExprAST> ParsePrimary() {
  switch (CurTok) {
  default:
    return LogError("unknown token when expecting an expression");
  case tok_identifier:
    return ParseIdentifierExpr();
  case tok_number:
    return ParseNumberExpr();
  case '(':
    return ParseParenExpr();
  case tok_if:
    return ParseIfExpr();
  case tok_for:
    return ParseForExpr();
  case tok_var:
    return ParseVarExpr();
  }
}
```

Tiếp theo ta định nghĩa ParseVarExpr:

```c++
/// varexpr ::= 'var' identifier ('=' expression)?
//                    (',' identifier ('=' expression)?)* 'in' expression
static std::unique_ptr<ExprAST> ParseVarExpr() {
  getNextToken();  // bỏ qua var.

  std::vector<std::pair<std::string, std::unique_ptr<ExprAST>>> VarNames;

  // Cần ít nhất một tên biến.
  if (CurTok != tok_identifier)
    return LogError("expected identifier after var");
```

Phần đầu của đoạn mã này phân tích danh sách các cặp identifier/expr vào vector cục bộ `VarNames`.

```c++
while (true) {
  std::string Name = IdentifierStr;
  getNextToken();  // bỏ qua identifier.

  // Đọc phần khởi tạo tùy chọn.
  std::unique_ptr<ExprAST> Init;
  if (CurTok == '=') {
    getNextToken(); // bỏ qua '='.

    Init = ParseExpression();
    if (!Init) return nullptr;
  }

  VarNames.push_back(std::make_pair(Name, std::move(Init)));

  // Kết thúc danh sách var, thoát vòng lặp.
  if (CurTok != ',') break;
  getNextToken(); // bỏ qua ','.

  if (CurTok != tok_identifier)
    return LogError("expected identifier list after var");
}
```

Khi đã phân tích xong tất cả các biến, ta phân tích phần thân và tạo nút AST:

```c++
  // Tại điểm này, bắt buộc phải có 'in'.
  if (CurTok != tok_in)
    return LogError("expected 'in' keyword after 'var'");
  getNextToken();  // bỏ qua 'in'.

  auto Body = ParseExpression();
  if (!Body)
    return nullptr;

  return std::make_unique<VarExprAST>(std::move(VarNames),
                                       std::move(Body));
}
```

Giờ ta đã có thể phân tích và biểu diễn mã, ta cần hỗ trợ sinh LLVM IR cho nó. Đoạn mã này bắt đầu như sau:

```c++
Value *VarExprAST::codegen() {
  std::vector<AllocaInst *> OldBindings;

  Function *TheFunction = Builder->GetInsertBlock()->getParent();

  // Đăng ký tất cả các biến và sinh mã cho phần khởi tạo của chúng.
  for (unsigned i = 0, e = VarNames.size(); i != e; ++i) {
    const std::string &VarName = VarNames[i].first;
    ExprAST *Init = VarNames[i].second.get();
```

Về cơ bản, nó lặp qua tất cả các biến và cài đặt từng biến một. Với mỗi biến được đưa vào bảng ký hiệu, ta ghi nhớ giá trị cũ bị thay thế vào OldBindings.

```c++
  // Sinh mã cho phần khởi tạo trước khi thêm biến vào phạm vi, điều này ngăn
  // phần khởi tạo tham chiếu tới chính biến đó, và cho phép những thứ
  // như sau:
  //  var a = 1 in
  //    var a = a in ...   # tham chiếu tới 'a' bên ngoài.
  Value *InitVal;
  if (Init) {
    InitVal = Init->codegen();
    if (!InitVal)
      return nullptr;
  } else { // Nếu không được chỉ định, dùng 0.0.
    InitVal = ConstantFP::get(*TheContext, APFloat(0.0));
  }

  AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, VarName);
  Builder->CreateStore(InitVal, Alloca);

  // Ghi nhớ ràng buộc biến cũ để có thể khôi phục lại khi
  // thoát khỏi đệ quy.
  OldBindings.push_back(NamedValues[VarName]);

  // Ghi nhớ ràng buộc này.
  NamedValues[VarName] = Alloca;
}
```

Ở đây chú thích còn nhiều hơn cả mã. Ý tưởng cơ bản là ta sinh mã cho phần khởi tạo, tạo alloca, rồi cập nhật bảng ký hiệu để trỏ tới nó. Khi tất cả các biến đã được cài đặt vào bảng ký hiệu, ta đánh giá phần thân của biểu thức var/in:

```c++
// Sinh mã cho phần thân, giờ khi tất cả các biến đều đã nằm trong phạm vi.
Value *BodyVal = Body->codegen();
if (!BodyVal)
  return nullptr;
```

Cuối cùng, trước khi trả về, ta khôi phục các ràng buộc biến trước đó:

```c++
  // Loại bỏ tất cả các biến của ta khỏi phạm vi.
  for (unsigned i = 0, e = VarNames.size(); i != e; ++i)
    NamedValues[VarNames[i].first] = OldBindings[i];

  // Trả về kết quả tính toán của phần thân.
  return BodyVal;
}
```

Kết quả cuối cùng của tất cả những điều này là ta có được các định nghĩa biến với phạm vi đúng đắn, và thậm chí còn cho phép (một cách hiển nhiên) thay đổi giá trị của chúng :).

Như vậy, ta đã hoàn thành những gì đặt ra. Ví dụ fib dạng lặp xinh xắn trong phần giới thiệu được biên dịch và chạy rất tốt. Pass mem2reg tối ưu hóa tất cả các biến trên stack của ta thành thanh ghi SSA, chèn nút PHI khi cần, và front-end của ta vẫn đơn giản: không thấy bóng dáng phép tính "iterated dominance frontier" nào cả.

## Toàn bộ mã nguồn

Dưới đây là toàn bộ mã nguồn cho ví dụ xuyên suốt của chúng ta, được mở rộng với biến khả biến và hỗ trợ var/in. Để build ví dụ này, hãy dùng:

```bash
# Biên dịch
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
# Chạy
./toy
```

Đây là mã nguồn:

```{literalinclude} ../../../examples/Kaleidoscope/Chapter7/toy.cpp
:language: c++
```

[Tiếp theo: Biên dịch ra mã đối tượng](LangImpl08.md)
