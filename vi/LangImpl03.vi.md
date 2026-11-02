# Kaleidoscope: Sinh mã LLVM IR

## Giới thiệu Chương 3

Chào mừng bạn đến với Chương 3 của hướng dẫn "[Hiện thực một ngôn ngữ với LLVM](index.md)". Chương này hướng dẫn bạn cách biến đổi [Cây cú pháp trừu tượng (Abstract Syntax Tree)](LangImpl02.md) đã xây dựng ở Chương 2 thành LLVM IR. Qua đó, bạn sẽ hiểu thêm đôi chút về cách LLVM vận hành, đồng thời thấy được LLVM dễ sử dụng đến mức nào. Xây dựng lexer và parser tốn công hơn nhiều so với sinh mã LLVM IR. :)

**Xin lưu ý**: mã nguồn trong chương này và các chương sau yêu cầu LLVM 3.7 trở lên. LLVM 3.6 trở về trước sẽ không hoạt động. Ngoài ra, bạn cần dùng phiên bản hướng dẫn khớp với bản phát hành LLVM của mình: nếu đang dùng một bản phát hành chính thức của LLVM, hãy dùng phiên bản tài liệu đi kèm bản phát hành đó hoặc trên [trang các bản phát hành của llvm.org](https://llvm.org/releases/).

## Thiết lập cho việc sinh mã

Để sinh LLVM IR, chúng ta cần một vài thiết lập đơn giản để bắt đầu. Trước tiên, ta định nghĩa các phương thức sinh mã (codegen) ảo trong mỗi lớp AST:

```c++
/// ExprAST - Lớp cơ sở cho mọi nút biểu thức.
class ExprAST {
public:
  virtual ~ExprAST() = default;
  virtual Value *codegen() = 0;
};

/// NumberExprAST - Lớp biểu thức cho các literal số như "1.0".
class NumberExprAST : public ExprAST {
  double Val;

public:
  NumberExprAST(double Val) : Val(Val) {}
  Value *codegen() override;
};
...
```

Phương thức codegen() có nhiệm vụ phát sinh (emit) IR cho nút AST đó cùng với mọi thứ mà nó phụ thuộc vào, và tất cả đều trả về một đối tượng LLVM Value. "Value" là lớp dùng để biểu diễn một "[thanh ghi Static Single Assignment (SSA)](http://en.wikipedia.org/wiki/Static_single_assignment_form)" hay "giá trị SSA" trong LLVM. Đặc điểm nổi bật nhất của giá trị SSA là giá trị của nó được tính khi lệnh tương ứng thực thi, và nó không nhận giá trị mới cho đến khi (và nếu) lệnh đó được thực thi lại. Nói cách khác, không có cách nào để "thay đổi" một giá trị SSA. Để biết thêm thông tin, hãy tìm đọc về [Static Single Assignment](http://en.wikipedia.org/wiki/Static_single_assignment_form) – các khái niệm này thực ra khá tự nhiên một khi bạn đã thấm chúng.

Lưu ý rằng thay vì thêm các phương thức ảo vào hệ phân cấp lớp ExprAST, việc dùng [mẫu visitor (visitor pattern)](http://en.wikipedia.org/wiki/Visitor_pattern) hoặc một cách mô hình hóa khác cũng hợp lý. Một lần nữa, hướng dẫn này sẽ không đi sâu vào các thực hành kỹ nghệ phần mềm tốt: với mục đích của chúng ta, thêm một phương thức ảo là cách đơn giản nhất.

Thứ hai, chúng ta cần một phương thức "LogError" giống như đã dùng cho parser, dùng để báo các lỗi phát hiện trong quá trình sinh mã (ví dụ: sử dụng một tham số chưa được khai báo):

```c++
static std::unique_ptr<LLVMContext> TheContext;
static std::unique_ptr<IRBuilder<>> Builder;
static std::unique_ptr<Module> TheModule;
static std::map<std::string, Value *> NamedValues;

Value *LogErrorV(const char *Str) {
  LogError(Str);
  return nullptr;
}
```

Các biến tĩnh này sẽ được dùng trong quá trình sinh mã. `TheContext` là một đối tượng mờ (opaque) sở hữu nhiều cấu trúc dữ liệu cốt lõi của LLVM, chẳng hạn như bảng kiểu và bảng giá trị hằng. Chúng ta không cần hiểu chi tiết về nó, chỉ cần một thể hiện (instance) duy nhất để truyền vào các API yêu cầu nó.

Đối tượng `Builder` là một đối tượng trợ giúp, giúp việc sinh các lệnh LLVM trở nên dễ dàng. Các thể hiện của class template [IRBuilder](https://llvm.org/doxygen/IRBuilder_8h_source.html) theo dõi vị trí hiện tại để chèn lệnh và cung cấp các phương thức để tạo lệnh mới.

`TheModule` là một cấu trúc của LLVM chứa các hàm và biến toàn cục. Theo nhiều nghĩa, đây là cấu trúc cấp cao nhất mà LLVM IR dùng để chứa mã. Nó sẽ sở hữu bộ nhớ của toàn bộ IR mà chúng ta sinh ra, đó là lý do phương thức codegen() trả về một con trỏ thô `Value*` thay vì `unique_ptr<Value>`.

Map `NamedValues` theo dõi những giá trị nào được định nghĩa trong phạm vi (scope) hiện tại và biểu diễn LLVM của chúng là gì. (Nói cách khác, nó là bảng ký hiệu (symbol table) cho mã.) Trong phiên bản Kaleidoscope này, thứ duy nhất có thể được tham chiếu là các tham số hàm. Do đó, các tham số hàm sẽ nằm trong map này khi sinh mã cho thân hàm của chúng.

Với những nền tảng cơ bản này, chúng ta có thể bắt đầu bàn về cách sinh mã cho từng biểu thức. Lưu ý rằng điều này giả định `Builder` đã được thiết lập để sinh mã *vào* một nơi nào đó. Hiện tại, ta sẽ giả định việc này đã được thực hiện và chỉ dùng nó để phát sinh mã.

## Sinh mã cho biểu thức

Sinh mã LLVM cho các nút biểu thức rất đơn giản: chưa đến 45 dòng mã (kể cả chú thích) cho cả bốn loại nút biểu thức của chúng ta. Đầu tiên là các literal số:

```c++
Value *NumberExprAST::codegen() {
  return ConstantFP::get(*TheContext, APFloat(Val));
}
```

Trong LLVM IR, hằng số được biểu diễn bằng lớp `ConstantFP`, lớp này lưu giá trị số bên trong một `APFloat` (`APFloat` có khả năng lưu các hằng số dấu phẩy động với độ chính xác tùy ý — Arbitrary Precision). Đoạn mã này về cơ bản chỉ tạo và trả về một `ConstantFP`. Lưu ý rằng trong LLVM IR, các hằng đều được duy nhất hóa (uniqued) và dùng chung. Vì lý do này, API sử dụng thành ngữ "foo::get(...)" thay vì "new foo(..)" hay "foo::Create(..)".

```c++
Value *VariableExprAST::codegen() {
  // Tra cứu biến này trong hàm.
  Value *V = NamedValues[Name];
  if (!V)
    LogErrorV("Unknown variable name");
  return V;
}
```

Tham chiếu đến biến cũng khá đơn giản khi dùng LLVM. Trong phiên bản đơn giản của Kaleidoscope, ta giả định rằng biến đã được phát sinh ở đâu đó và giá trị của nó đã sẵn có. Trên thực tế, các giá trị duy nhất có thể nằm trong map `NamedValues` là các đối số của hàm. Đoạn mã này chỉ đơn giản kiểm tra xem tên được chỉ định có trong map hay không (nếu không, tức là đang tham chiếu đến một biến không xác định) và trả về giá trị tương ứng. Ở các chương sau, chúng ta sẽ bổ sung hỗ trợ cho [biến cảm ứng của vòng lặp (loop induction variable)](LangImpl05.md#for-loop-expression) trong bảng ký hiệu, và cho [biến cục bộ](LangImpl07.md#user-defined-local-variables).

```c++
Value *BinaryExprAST::codegen() {
  Value *L = LHS->codegen();
  Value *R = RHS->codegen();
  if (!L || !R)
    return nullptr;

  switch (Op) {
  case '+':
    return Builder->CreateFAdd(L, R, "addtmp");
  case '-':
    return Builder->CreateFSub(L, R, "subtmp");
  case '*':
    return Builder->CreateFMul(L, R, "multmp");
  case '<':
    L = Builder->CreateFCmpULT(L, R, "cmptmp");
    // Chuyển bool 0/1 thành double 0.0 hoặc 1.0
    return Builder->CreateUIToFP(L, Type::getDoubleTy(*TheContext),
                                 "booltmp");
  default:
    return LogErrorV("invalid binary operator");
  }
}
```

Các toán tử hai ngôi bắt đầu thú vị hơn. Ý tưởng cơ bản ở đây là ta đệ quy phát sinh mã cho vế trái của biểu thức, rồi đến vế phải, sau đó tính kết quả của biểu thức hai ngôi. Trong đoạn mã này, ta dùng một câu lệnh switch đơn giản trên mã toán tử (opcode) để tạo ra lệnh LLVM phù hợp.

Trong ví dụ trên, lớp builder của LLVM bắt đầu cho thấy giá trị của nó. IRBuilder biết vị trí cần chèn lệnh mới tạo, bạn chỉ cần chỉ định lệnh cần tạo (ví dụ với `CreateFAdd`), các toán hạng sử dụng (ở đây là `L` và `R`) và tùy chọn đặt tên cho lệnh được sinh ra.

Một điểm hay của LLVM là tên chỉ mang tính gợi ý. Chẳng hạn, nếu đoạn mã trên phát sinh nhiều biến "addtmp", LLVM sẽ tự động gắn cho mỗi biến một hậu tố số tăng dần và duy nhất. Tên giá trị cục bộ cho các lệnh hoàn toàn không bắt buộc, nhưng chúng giúp việc đọc các bản kết xuất (dump) IR dễ dàng hơn nhiều.

[Các lệnh LLVM](../../LangRef.md#instruction-reference) bị ràng buộc bởi các quy tắc nghiêm ngặt: ví dụ, toán hạng trái và phải của [lệnh add](../../LangRef.md#add-instruction) phải cùng kiểu, và kiểu kết quả của phép add phải khớp với kiểu của các toán hạng. Vì mọi giá trị trong Kaleidoscope đều là double, mã cho add, sub và mul trở nên rất đơn giản.

Mặt khác, LLVM quy định rằng [lệnh fcmp](../../LangRef.md#fcmp-instruction) luôn trả về giá trị 'i1' (số nguyên một bit). Vấn đề là Kaleidoscope muốn giá trị đó là 0.0 hoặc 1.0. Để có được ngữ nghĩa này, ta kết hợp lệnh fcmp với [lệnh uitofp](../../LangRef.md#uitofp-to-instruction). Lệnh này chuyển số nguyên đầu vào thành giá trị dấu phẩy động bằng cách coi đầu vào là một giá trị không dấu. Ngược lại, nếu dùng [lệnh sitofp](../../LangRef.md#sitofp-to-instruction), toán tử `<` của Kaleidoscope sẽ trả về 0.0 và -1.0, tùy thuộc vào giá trị đầu vào.

```c++
Value *CallExprAST::codegen() {
  // Tra cứu tên trong bảng toàn cục của module.
  Function *CalleeF = TheModule->getFunction(Callee);
  if (!CalleeF)
    return LogErrorV("Unknown function referenced");

  // Báo lỗi nếu số đối số không khớp.
  if (CalleeF->arg_size() != Args.size())
    return LogErrorV("Incorrect # arguments passed");

  std::vector<Value *> ArgsV;
  for (unsigned i = 0, e = Args.size(); i != e; ++i) {
    ArgsV.push_back(Args[i]->codegen());
    if (!ArgsV.back())
      return nullptr;
  }

  return Builder->CreateCall(CalleeF, ArgsV, "calltmp");
}
```

Sinh mã cho lời gọi hàm khá đơn giản với LLVM. Đoạn mã trên trước tiên tra cứu tên hàm trong bảng ký hiệu của LLVM Module. Hãy nhớ rằng LLVM Module là vùng chứa các hàm mà chúng ta đang JIT. Bằng cách đặt cho mỗi hàm cùng tên với tên mà người dùng chỉ định, ta có thể dùng bảng ký hiệu của LLVM để phân giải tên hàm.

Khi đã có hàm cần gọi, ta đệ quy sinh mã cho từng đối số sẽ được truyền vào, rồi tạo một [lệnh call](../../LangRef.md#call-instruction) của LLVM. Lưu ý rằng mặc định LLVM dùng quy ước gọi hàm (calling convention) C gốc, cho phép các lời gọi này gọi được cả các hàm thư viện chuẩn như "sin" và "cos" mà không cần thêm công sức nào.

Đến đây là kết thúc phần xử lý bốn loại biểu thức cơ bản mà Kaleidoscope hiện có. Bạn cứ thoải mái bổ sung thêm. Ví dụ, khi duyệt qua [tài liệu tham khảo ngôn ngữ LLVM](../../LangRef.md), bạn sẽ thấy một số lệnh thú vị khác rất dễ tích hợp vào khung cơ bản của chúng ta.

## Sinh mã cho hàm

Việc sinh mã cho prototype và hàm phải xử lý khá nhiều chi tiết, khiến mã của chúng kém đẹp hơn so với sinh mã cho biểu thức, nhưng lại cho phép chúng ta minh họa một số điểm quan trọng. Trước tiên, hãy nói về việc sinh mã cho prototype: chúng được dùng cho cả thân hàm lẫn khai báo hàm bên ngoài (external). Đoạn mã bắt đầu như sau:

```c++
Function *PrototypeAST::codegen() {
  // Tạo kiểu hàm:  double(double,double) v.v.
  std::vector<Type*> Doubles(Args.size(),
                             Type::getDoubleTy(*TheContext));
  FunctionType *FT =
    FunctionType::get(Type::getDoubleTy(*TheContext), Doubles, false);

  Function *F =
    Function::Create(FT, Function::ExternalLinkage, Name, TheModule.get());
```

Đoạn mã này gói gọn rất nhiều sức mạnh trong vài dòng. Trước hết, lưu ý rằng hàm này trả về `Function*` thay vì `Value*`. Vì một "prototype" thực chất mô tả giao diện bên ngoài của một hàm (chứ không phải giá trị được tính bởi một biểu thức), nên hợp lý khi nó trả về LLVM Function tương ứng sau khi được sinh mã.

Lời gọi `FunctionType::get` tạo ra `FunctionType` cần dùng cho một Prototype nhất định. Vì mọi đối số hàm trong Kaleidoscope đều có kiểu double, dòng đầu tiên tạo một vector gồm "N" kiểu double của LLVM. Sau đó nó dùng phương thức `Functiontype::get` để tạo một kiểu hàm nhận "N" giá trị double làm đối số, trả về một double làm kết quả và không phải là vararg (tham số false biểu thị điều này). Lưu ý rằng các Type trong LLVM cũng được duy nhất hóa giống như Constant, vì vậy bạn không "new" một kiểu mà "get" nó.

Dòng cuối cùng ở trên thực sự tạo ra IR Function tương ứng với Prototype. Nó chỉ định kiểu, liên kết (linkage) và tên cần dùng, cũng như module để chèn hàm vào. "[Liên kết ngoài (external linkage)](../../LangRef.md#linkage-types)" nghĩa là hàm có thể được định nghĩa bên ngoài module hiện tại và/hoặc có thể được gọi bởi các hàm bên ngoài module. Name được truyền vào là tên do người dùng chỉ định: vì "`TheModule`" được chỉ định, tên này sẽ được đăng ký trong bảng ký hiệu của "`TheModule`".

```c++
// Đặt tên cho tất cả các đối số.
unsigned Idx = 0;
for (auto &Arg : F->args())
  Arg.setName(Args[Idx++]);

return F;
```

Cuối cùng, ta đặt tên cho từng đối số của hàm theo các tên được cho trong Prototype. Bước này không thực sự bắt buộc, nhưng giữ các tên nhất quán giúp IR dễ đọc hơn, đồng thời cho phép mã phía sau tham chiếu trực tiếp đến các đối số theo tên của chúng, thay vì phải tra cứu trong Prototype AST.

Đến đây, ta có một prototype hàm chưa có thân. Đây là cách LLVM IR biểu diễn khai báo hàm. Đối với các câu lệnh extern trong Kaleidoscope, chừng này là đủ. Tuy nhiên, với định nghĩa hàm, ta cần sinh mã và gắn thêm thân hàm.

```c++
Function *FunctionAST::codegen() {
    // Trước tiên, kiểm tra xem đã có hàm từ một khai báo 'extern' trước đó chưa.
  Function *TheFunction = TheModule->getFunction(Proto->getName());

  if (!TheFunction)
    TheFunction = Proto->codegen();

  if (!TheFunction)
    return nullptr;

  if (!TheFunction->empty())
    return (Function*)LogErrorV("Function cannot be redefined.");
```

Với định nghĩa hàm, ta bắt đầu bằng việc tìm trong bảng ký hiệu của TheModule xem đã có phiên bản nào của hàm này chưa, phòng trường hợp nó đã được tạo bằng một câu lệnh 'extern'. Nếu Module::getFunction trả về null thì chưa có phiên bản nào trước đó, nên ta sẽ sinh mã một phiên bản từ Prototype. Trong cả hai trường hợp, ta muốn khẳng định (assert) rằng hàm đang rỗng (tức là chưa có thân) trước khi bắt đầu.

```c++
// Tạo một basic block mới để bắt đầu chèn lệnh vào.
BasicBlock *BB = BasicBlock::Create(*TheContext, "entry", TheFunction);
Builder->SetInsertPoint(BB);

// Ghi lại các đối số của hàm vào map NamedValues.
NamedValues.clear();
for (auto &Arg : TheFunction->args())
  NamedValues[std::string(Arg.getName())] = &Arg;
```

Giờ ta đến bước thiết lập `Builder`. Dòng đầu tiên tạo một [basic block](http://en.wikipedia.org/wiki/Basic_block) mới (tên là "entry"), được chèn vào `TheFunction`. Dòng thứ hai sau đó báo cho builder biết rằng các lệnh mới cần được chèn vào cuối basic block mới này. Basic block trong LLVM là một phần quan trọng của hàm, chúng định nghĩa nên [Đồ thị luồng điều khiển (Control Flow Graph)](http://en.wikipedia.org/wiki/Control_flow_graph). Vì chưa có luồng điều khiển nào, các hàm của chúng ta hiện chỉ chứa một block. Chúng ta sẽ khắc phục điều này ở [Chương 5](LangImpl05.md) :).

Tiếp theo, ta thêm các đối số của hàm vào map NamedValues (sau khi xóa sạch nó) để các nút `VariableExprAST` có thể truy cập chúng.

```c++
if (Value *RetVal = Body->codegen()) {
  // Hoàn tất hàm.
  Builder->CreateRet(RetVal);

  // Kiểm tra mã được sinh ra, đảm bảo tính nhất quán.
  verifyFunction(*TheFunction);

  return TheFunction;
}
```

Khi điểm chèn đã được thiết lập và map NamedValues đã được điền, ta gọi phương thức `codegen()` cho biểu thức gốc của hàm. Nếu không có lỗi, lời gọi này sẽ phát sinh mã tính toán biểu thức vào entry block và trả về giá trị đã tính. Giả sử không có lỗi, ta tạo một [lệnh ret](../../LangRef.md#ret-instruction) của LLVM để hoàn tất hàm. Khi hàm đã được xây dựng xong, ta gọi `verifyFunction`, một hàm do LLVM cung cấp. Hàm này thực hiện nhiều kiểm tra tính nhất quán khác nhau trên mã được sinh ra, nhằm xác định xem trình biên dịch của chúng ta có làm đúng mọi thứ hay không. Việc dùng nó rất quan trọng: nó có thể bắt được rất nhiều lỗi. Khi hàm đã hoàn tất và được kiểm tra, ta trả về nó.

```c++
  // Lỗi khi đọc thân hàm, xóa hàm.
  TheFunction->eraseFromParent();
  return nullptr;
}
```

Phần duy nhất còn lại là xử lý trường hợp lỗi. Để đơn giản, ta xử lý bằng cách chỉ xóa hàm đã tạo bằng phương thức `eraseFromParent`. Điều này cho phép người dùng định nghĩa lại một hàm mà trước đó họ đã gõ sai: nếu không xóa, hàm đó sẽ tồn tại trong bảng ký hiệu cùng với thân hàm, ngăn cản việc định nghĩa lại về sau.

Tuy vậy, đoạn mã này có một lỗi: nếu phương thức `FunctionAST::codegen()` tìm thấy một IR Function đã tồn tại, nó không kiểm tra chữ ký (signature) của hàm đó với prototype của chính định nghĩa. Điều này có nghĩa là một khai báo 'extern' trước đó sẽ được ưu tiên hơn chữ ký của định nghĩa hàm, có thể khiến việc sinh mã thất bại, chẳng hạn khi các đối số của hàm được đặt tên khác nhau. Có nhiều cách để sửa lỗi này, hãy thử xem bạn nghĩ ra được cách nào! Đây là một testcase:

```
extern foo(a);     # ok, định nghĩa foo.
def foo(b) b;      # Lỗi: Unknown variable name. (khai báo dùng 'a' được ưu tiên).
```

## Thay đổi ở driver và lời kết

Hiện tại, việc sinh mã LLVM chưa mang lại cho chúng ta nhiều lợi ích, ngoại trừ việc có thể xem các lời gọi IR đẹp mắt. Mã mẫu chèn các lời gọi codegen vào các hàm "`HandleDefinition`", "`HandleExtern`", v.v., rồi kết xuất LLVM IR ra. Điều này cho ta một cách hay để xem LLVM IR của các hàm đơn giản. Ví dụ:

```
ready> 4+5;
Read top-level expression:
define double @__anon_expr() {
entry:
  ret double 9.000000e+00
}
```

Hãy để ý cách parser biến biểu thức cấp cao nhất thành các hàm ẩn danh cho chúng ta. Điều này sẽ rất tiện khi ta [thêm hỗ trợ JIT](LangImpl04.md#adding-a-jit-compiler) ở chương sau. Cũng lưu ý rằng mã được chuyển đổi rất sát nghĩa đen, không có tối ưu hóa nào được thực hiện ngoại trừ việc gộp hằng (constant folding) đơn giản do IRBuilder đảm nhận. Chúng ta sẽ [thêm các tối ưu hóa](LangImpl04.md#trivial-constant-folding) một cách tường minh ở chương sau.

```
ready> def foo(a b) a*a + 2*a*b + b*b;
Read function definition:
define double @foo(double %a, double %b) {
entry:
  %multmp = fmul double %a, %a
  %multmp1 = fmul double 2.000000e+00, %a
  %multmp2 = fmul double %multmp1, %b
  %addtmp = fadd double %multmp, %multmp2
  %multmp3 = fmul double %b, %b
  %addtmp4 = fadd double %addtmp, %multmp3
  ret double %addtmp4
}
```

Ví dụ này cho thấy một số phép tính số học đơn giản. Hãy để ý sự tương đồng rõ rệt với các lời gọi LLVM builder mà ta dùng để tạo lệnh.

```
ready> def bar(a) foo(a, 4.0) + bar(31337);
Read function definition:
define double @bar(double %a) {
entry:
  %calltmp = call double @foo(double %a, double 4.000000e+00)
  %calltmp1 = call double @bar(double 3.133700e+04)
  %addtmp = fadd double %calltmp, %calltmp1
  ret double %addtmp
}
```

Ví dụ này cho thấy một số lời gọi hàm. Lưu ý rằng hàm này sẽ mất rất nhiều thời gian để thực thi nếu bạn gọi nó. Về sau, chúng ta sẽ thêm luồng điều khiển có điều kiện để đệ quy thực sự trở nên hữu ích :).

```
ready> extern cos(x);
Read extern:
declare double @cos(double)

ready> cos(1.234);
Read top-level expression:
define double @__anon_expr() {
entry:
  %calltmp = call double @cos(double 1.234000e+00)
  ret double %calltmp
}
```

Ví dụ này cho thấy một khai báo extern cho hàm "cos" của libm và một lời gọi đến nó.

:::{todo}
Loại bỏ lexer `llvm` tệ hại của Pygments. Nó hoàn toàn bỏ cuộc trong việc tô sáng đoạn này chỉ vì dòng đầu tiên.
:::

```
ready> ^D
; ModuleID = 'my cool jit'

define double @0() {
entry:
  %addtmp = fadd double 4.000000e+00, 5.000000e+00
  ret double %addtmp
}

define double @foo(double %a, double %b) {
entry:
  %multmp = fmul double %a, %a
  %multmp1 = fmul double 2.000000e+00, %a
  %multmp2 = fmul double %multmp1, %b
  %addtmp = fadd double %multmp, %multmp2
  %multmp3 = fmul double %b, %b
  %addtmp4 = fadd double %addtmp, %multmp3
  ret double %addtmp4
}

define double @bar(double %a) {
entry:
  %calltmp = call double @foo(double %a, double 4.000000e+00)
  %calltmp1 = call double @bar(double 3.133700e+04)
  %addtmp = fadd double %calltmp, %calltmp1
  ret double %addtmp
}

declare double @cos(double)

define double @1() {
entry:
  %calltmp = call double @cos(double 1.234000e+00)
  ret double %calltmp
}
```

Khi bạn thoát khỏi bản demo hiện tại (bằng cách gửi EOF qua CTRL+D trên Linux hoặc CTRL+Z rồi ENTER trên Windows), nó sẽ kết xuất IR cho toàn bộ module đã sinh ra. Tại đây bạn có thể thấy bức tranh tổng thể với tất cả các hàm tham chiếu lẫn nhau.

Đến đây là kết thúc chương thứ ba của hướng dẫn Kaleidoscope. Tiếp theo, chúng ta sẽ mô tả cách [thêm hỗ trợ sinh mã JIT và bộ tối ưu hóa](LangImpl04.md) để có thể thực sự bắt đầu chạy mã!

## Mã nguồn đầy đủ

Dưới đây là toàn bộ mã nguồn cho ví dụ xuyên suốt của chúng ta, đã được bổ sung bộ sinh mã LLVM. Vì sử dụng các thư viện LLVM, chúng ta cần liên kết (link) chúng vào. Để làm điều này, ta dùng công cụ [llvm-config](https://llvm.org/cmds/llvm-config.html) để cho makefile/dòng lệnh biết cần dùng những tùy chọn nào:

```bash
# Biên dịch
clang++ -g -O3 toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core` -o toy
# Chạy
./toy
```

Đây là mã nguồn:

```{literalinclude} ../../../examples/Kaleidoscope/Chapter3/toy.cpp
:language: c++
```

[Tiếp theo: Thêm hỗ trợ JIT và bộ tối ưu hóa](LangImpl04.md)
