# Kaleidoscope: Mở rộng ngôn ngữ: Toán tử do người dùng định nghĩa

## Giới thiệu chương 6

Chào mừng bạn đến với chương 6 của loạt hướng dẫn "[Hiện thực một ngôn ngữ với LLVM](index.md)". Đến thời điểm này, chúng ta đã có một ngôn ngữ hoạt động đầy đủ, khá tối giản nhưng cũng hữu ích. Tuy nhiên, nó vẫn còn một vấn đề lớn: ngôn ngữ của chúng ta không có nhiều toán tử hữu ích (như phép chia, phủ định logic, hay thậm chí bất kỳ phép so sánh nào ngoài phép nhỏ hơn).

Chương này của loạt hướng dẫn sẽ rẽ ngang khá táo bạo để bổ sung các toán tử do người dùng định nghĩa vào ngôn ngữ Kaleidoscope vốn đơn giản và đẹp đẽ. Sự rẽ ngang này khiến ngôn ngữ của chúng ta trở nên đơn giản và xấu xí ở một vài khía cạnh, nhưng đồng thời cũng mạnh mẽ hơn. Một trong những điều tuyệt vời khi tự tạo ngôn ngữ của riêng mình là bạn được quyết định điều gì là tốt hay xấu. Trong hướng dẫn này, chúng ta sẽ mặc định rằng việc dùng nó như một cách để trình bày một số kỹ thuật phân tích cú pháp (parsing) thú vị là chấp nhận được.

Ở cuối hướng dẫn này, chúng ta sẽ đi qua một ứng dụng Kaleidoscope mẫu [vẽ tập Mandelbrot](#kicking-the-tires). Đây là một ví dụ về những gì bạn có thể xây dựng với Kaleidoscope và tập tính năng của nó.

## Toán tử do người dùng định nghĩa: Ý tưởng

"Nạp chồng toán tử" (operator overloading) mà chúng ta sẽ thêm vào Kaleidoscope tổng quát hơn so với các ngôn ngữ như C++. Trong C++, bạn chỉ được phép định nghĩa lại các toán tử đã có: bạn không thể thay đổi ngữ pháp bằng lập trình, đưa vào toán tử mới, thay đổi mức độ ưu tiên, v.v. Trong chương này, chúng ta sẽ thêm khả năng đó vào Kaleidoscope, cho phép người dùng tự hoàn thiện tập toán tử được hỗ trợ.

Mục đích của việc đi sâu vào toán tử do người dùng định nghĩa trong một hướng dẫn như thế này là để cho thấy sức mạnh và sự linh hoạt của việc dùng một bộ phân tích cú pháp (parser) viết tay. Cho đến giờ, parser mà chúng ta đã hiện thực sử dụng phương pháp đệ quy đi xuống (recursive descent) cho phần lớn ngữ pháp và phân tích theo độ ưu tiên toán tử (operator precedence parsing) cho các biểu thức. Xem [chương 2](LangImpl02.md) để biết chi tiết. Nhờ sử dụng phân tích theo độ ưu tiên toán tử, rất dễ để cho phép lập trình viên đưa toán tử mới vào ngữ pháp: ngữ pháp có thể được mở rộng động ngay khi JIT đang chạy.

Hai tính năng cụ thể mà chúng ta sẽ thêm là các toán tử một ngôi (unary) có thể lập trình được (hiện tại Kaleidoscope hoàn toàn chưa có toán tử một ngôi nào) cũng như các toán tử hai ngôi (binary). Ví dụ:

```
# Phủ định logic một ngôi (not).
def unary!(v)
  if v then
    0
  else
    1;

# Định nghĩa > với cùng độ ưu tiên như <.
def binary> 10 (LHS RHS)
  RHS < LHS;

# "Hoặc logic" hai ngôi (lưu ý rằng nó không "đoản mạch" (short circuit))
def binary| 5 (LHS RHS)
  if LHS then
    1
  else if RHS then
    1
  else
    0;

# Định nghĩa = với độ ưu tiên thấp hơn một chút so với các toán tử quan hệ.
def binary= 9 (LHS RHS)
  !(LHS < RHS | LHS > RHS);
```

Nhiều ngôn ngữ mong muốn có thể hiện thực thư viện runtime chuẩn của mình bằng chính ngôn ngữ đó. Trong Kaleidoscope, chúng ta có thể hiện thực những phần đáng kể của ngôn ngữ ngay trong thư viện!

Chúng ta sẽ chia việc hiện thực các tính năng này thành hai phần: hiện thực hỗ trợ toán tử hai ngôi do người dùng định nghĩa và thêm toán tử một ngôi.

## Toán tử hai ngôi do người dùng định nghĩa

Việc thêm hỗ trợ cho toán tử hai ngôi do người dùng định nghĩa khá đơn giản với khung sườn hiện tại. Trước tiên, chúng ta sẽ thêm hỗ trợ cho các từ khóa unary/binary:

```c++
enum Token {
  ...
  // toán tử
  tok_binary = -11,
  tok_unary = -12
};
...
static int gettok() {
...
    if (IdentifierStr == "for")
      return tok_for;
    if (IdentifierStr == "in")
      return tok_in;
    if (IdentifierStr == "binary")
      return tok_binary;
    if (IdentifierStr == "unary")
      return tok_unary;
    return tok_identifier;
```

Đoạn này chỉ đơn giản thêm hỗ trợ cho các từ khóa unary và binary vào lexer, giống như chúng ta đã làm trong [các chương trước](LangImpl05.md#lexer-extensions-for-if-then-else). Một điểm hay của AST hiện tại là chúng ta biểu diễn các toán tử hai ngôi một cách hoàn toàn tổng quát bằng cách dùng mã ASCII của chúng làm opcode. Với các toán tử mở rộng, chúng ta sẽ dùng cùng cách biểu diễn này, vì vậy không cần thêm hỗ trợ mới nào cho AST hay parser.

Mặt khác, chúng ta phải có khả năng biểu diễn định nghĩa của những toán tử mới này, ở phần "def binary| 5" của định nghĩa hàm. Trong ngữ pháp hiện tại, "tên" của định nghĩa hàm được phân tích theo luật sinh (production) "prototype" và đưa vào nút AST `PrototypeAST`. Để biểu diễn các toán tử do người dùng định nghĩa dưới dạng prototype, chúng ta phải mở rộng nút AST `PrototypeAST` như sau:

```c++
/// PrototypeAST - Lớp này biểu diễn "prototype" của một hàm,
/// lưu lại tên các đối số cũng như việc nó có phải là toán tử hay không.
class PrototypeAST {
  std::string Name;
  std::vector<std::string> Args;
  bool IsOperator;
  unsigned Precedence;  // Độ ưu tiên nếu là toán tử hai ngôi.

public:
  PrototypeAST(const std::string &Name, std::vector<std::string> Args,
               bool IsOperator = false, unsigned Prec = 0)
  : Name(Name), Args(std::move(Args)), IsOperator(IsOperator),
    Precedence(Prec) {}

  Function *codegen();
  const std::string &getName() const { return Name; }

  bool isUnaryOp() const { return IsOperator && Args.size() == 1; }
  bool isBinaryOp() const { return IsOperator && Args.size() == 2; }

  char getOperatorName() const {
    assert(isUnaryOp() || isBinaryOp());
    return Name[Name.size() - 1];
  }

  unsigned getBinaryPrecedence() const { return Precedence; }
};
```

Về cơ bản, ngoài việc biết tên của prototype, giờ đây chúng ta còn theo dõi xem nó có phải là toán tử hay không, và nếu có thì toán tử đó ở mức độ ưu tiên nào. Độ ưu tiên chỉ được dùng cho toán tử hai ngôi (như bạn sẽ thấy bên dưới, nó đơn giản là không áp dụng cho toán tử một ngôi). Giờ khi đã có cách biểu diễn prototype cho toán tử do người dùng định nghĩa, chúng ta cần phân tích cú pháp cho nó:

```c++
/// prototype
///   ::= id '(' id* ')'
///   ::= binary LETTER number? (id, id)
static std::unique_ptr<PrototypeAST> ParsePrototype() {
  std::string FnName;

  unsigned Kind = 0;  // 0 = định danh, 1 = một ngôi, 2 = hai ngôi.
  unsigned BinaryPrecedence = 30;

  switch (CurTok) {
  default:
    return LogErrorP("Expected function name in prototype");
  case tok_identifier:
    FnName = IdentifierStr;
    Kind = 0;
    getNextToken();
    break;
  case tok_binary:
    getNextToken();
    if (!isascii(CurTok))
      return LogErrorP("Expected binary operator");
    FnName = "binary";
    FnName += (char)CurTok;
    Kind = 2;
    getNextToken();

    // Đọc độ ưu tiên nếu có.
    if (CurTok == tok_number) {
      if (NumVal < 1 || NumVal > 100)
        return LogErrorP("Invalid precedence: must be 1..100");
      BinaryPrecedence = (unsigned)NumVal;
      getNextToken();
    }
    break;
  }

  if (CurTok != '(')
    return LogErrorP("Expected '(' in prototype");

  std::vector<std::string> ArgNames;
  while (getNextToken() == tok_identifier)
    ArgNames.push_back(IdentifierStr);
  if (CurTok != ')')
    return LogErrorP("Expected ')' in prototype");

  // thành công.
  getNextToken();  // tiêu thụ ')'.

  // Kiểm tra số lượng tên có đúng với toán tử hay không.
  if (Kind && ArgNames.size() != Kind)
    return LogErrorP("Invalid number of operands for operator");

  return std::make_unique<PrototypeAST>(FnName, std::move(ArgNames), Kind != 0,
                                         BinaryPrecedence);
}
```

Đây đều là mã phân tích cú pháp khá dễ hiểu, và chúng ta đã thấy rất nhiều đoạn mã tương tự trước đây. Một điểm thú vị trong đoạn mã trên là vài dòng thiết lập `FnName` cho toán tử hai ngôi. Chúng tạo ra các tên như "binary@" cho toán tử "@" mới được định nghĩa. Cách làm này tận dụng việc tên ký hiệu trong bảng ký hiệu (symbol table) của LLVM được phép chứa bất kỳ ký tự nào, kể cả ký tự nul nằm bên trong.

Điều thú vị tiếp theo cần thêm là hỗ trợ sinh mã (codegen) cho các toán tử hai ngôi này. Với cấu trúc hiện tại, đây chỉ là việc thêm một nhánh default đơn giản cho nút toán tử hai ngôi hiện có:

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
    break;
  }

  // Nếu đây không phải toán tử hai ngôi dựng sẵn thì nó phải là toán tử do người
  // dùng định nghĩa. Phát sinh lời gọi đến nó.
  Function *F = getFunction(std::string("binary") + Op);
  assert(F && "binary operator not found!");

  Value *Ops[2] = { L, R };
  return Builder->CreateCall(F, Ops, "binop");
}
```

Như bạn thấy ở trên, đoạn mã mới thực sự rất đơn giản. Nó chỉ tra cứu toán tử tương ứng trong bảng ký hiệu và sinh ra một lời gọi hàm đến nó. Vì các toán tử do người dùng định nghĩa được xây dựng như những hàm bình thường (bởi "prototype" rốt cuộc chỉ là một hàm có tên phù hợp), mọi thứ đều khớp vào đúng chỗ.

Mảnh mã cuối cùng chúng ta còn thiếu là một chút "phép thuật" ở cấp cao nhất (top-level):

```c++
Function *FunctionAST::codegen() {
  // Chuyển quyền sở hữu prototype sang map FunctionProtos, nhưng giữ lại
  // một tham chiếu đến nó để dùng bên dưới.
  auto &P = *Proto;
  FunctionProtos[Proto->getName()] = std::move(Proto);
  Function *TheFunction = getFunction(P.getName());
  if (!TheFunction)
    return nullptr;

  // Nếu đây là một toán tử, đăng ký nó.
  if (P.isBinaryOp())
    BinopPrecedence[P.getOperatorName()] = P.getBinaryPrecedence();

  // Tạo một basic block mới để bắt đầu chèn lệnh vào.
  BasicBlock *BB = BasicBlock::Create(*TheContext, "entry", TheFunction);
  ...
```

Về cơ bản, trước khi sinh mã cho một hàm, nếu đó là toán tử do người dùng định nghĩa, chúng ta đăng ký nó vào bảng độ ưu tiên. Điều này cho phép logic phân tích toán tử hai ngôi có sẵn xử lý được nó. Vì chúng ta đang làm việc với một parser theo độ ưu tiên toán tử hoàn toàn tổng quát, đây là tất cả những gì cần làm để "mở rộng ngữ pháp".

Giờ đây chúng ta đã có các toán tử hai ngôi do người dùng định nghĩa thực sự hữu ích. Phần này dựa rất nhiều vào khung sườn mà chúng ta đã xây dựng trước đó cho các toán tử khác. Việc thêm toán tử một ngôi thì thử thách hơn một chút, vì chúng ta chưa có bất kỳ khung sườn nào cho nó — hãy cùng xem cần những gì.

## Toán tử một ngôi do người dùng định nghĩa

Vì hiện tại ngôn ngữ Kaleidoscope chưa hỗ trợ toán tử một ngôi, chúng ta sẽ cần thêm mọi thứ để hỗ trợ chúng. Ở trên, chúng ta đã thêm hỗ trợ đơn giản cho từ khóa 'unary' vào lexer. Ngoài ra, chúng ta cần một nút AST:

```c++
/// UnaryExprAST - Lớp biểu thức cho toán tử một ngôi.
class UnaryExprAST : public ExprAST {
  char Opcode;
  std::unique_ptr<ExprAST> Operand;

public:
  UnaryExprAST(char Opcode, std::unique_ptr<ExprAST> Operand)
    : Opcode(Opcode), Operand(std::move(Operand)) {}

  Value *codegen() override;
};
```

Đến giờ thì nút AST này đã rất đơn giản và dễ hiểu. Nó phản chiếu trực tiếp nút AST của toán tử hai ngôi, chỉ khác là nó chỉ có một nút con. Với nút này, chúng ta cần thêm logic phân tích cú pháp. Phân tích một toán tử một ngôi khá đơn giản: chúng ta sẽ thêm một hàm mới để làm việc đó:

```c++
/// unary
///   ::= primary
///   ::= '!' unary
static std::unique_ptr<ExprAST> ParseUnary() {
  // Nếu token hiện tại không phải toán tử, nó phải là một biểu thức primary.
  if (!isascii(CurTok) || CurTok == '(' || CurTok == ',')
    return ParsePrimary();

  // Nếu đây là toán tử một ngôi, đọc nó.
  int Opc = CurTok;
  getNextToken();
  if (auto Operand = ParseUnary())
    return std::make_unique<UnaryExprAST>(Opc, std::move(Operand));
  return nullptr;
}
```

Ngữ pháp chúng ta thêm vào ở đây khá đơn giản. Nếu gặp một toán tử một ngôi khi phân tích một toán tử primary, chúng ta tiêu thụ toán tử đó như một tiền tố và phân tích phần còn lại như một toán tử một ngôi khác. Điều này cho phép chúng ta xử lý nhiều toán tử một ngôi liên tiếp (ví dụ "!!x"). Lưu ý rằng toán tử một ngôi không thể có cách phân tích mơ hồ như toán tử hai ngôi, vì vậy không cần thông tin về độ ưu tiên.

Vấn đề của hàm này là chúng ta cần gọi ParseUnary từ đâu đó. Để làm vậy, chúng ta sửa những nơi trước đây gọi ParsePrimary để chúng gọi ParseUnary thay thế:

```c++
/// binoprhs
///   ::= ('+' unary)*
static std::unique_ptr<ExprAST> ParseBinOpRHS(int ExprPrec,
                                              std::unique_ptr<ExprAST> LHS) {
  ...
    // Phân tích biểu thức một ngôi đứng sau toán tử hai ngôi.
    auto RHS = ParseUnary();
    if (!RHS)
      return nullptr;
  ...
}
/// expression
///   ::= unary binoprhs
///
static std::unique_ptr<ExprAST> ParseExpression() {
  auto LHS = ParseUnary();
  if (!LHS)
    return nullptr;

  return ParseBinOpRHS(0, std::move(LHS));
}
```

Với hai thay đổi đơn giản này, giờ đây chúng ta có thể phân tích toán tử một ngôi và xây dựng AST cho chúng. Tiếp theo, chúng ta cần thêm hỗ trợ trong parser cho prototype, để phân tích prototype của toán tử một ngôi. Chúng ta mở rộng đoạn mã cho toán tử hai ngôi ở trên như sau:

```c++
/// prototype
///   ::= id '(' id* ')'
///   ::= binary LETTER number? (id, id)
///   ::= unary LETTER (id)
static std::unique_ptr<PrototypeAST> ParsePrototype() {
  std::string FnName;

  unsigned Kind = 0;  // 0 = định danh, 1 = một ngôi, 2 = hai ngôi.
  unsigned BinaryPrecedence = 30;

  switch (CurTok) {
  default:
    return LogErrorP("Expected function name in prototype");
  case tok_identifier:
    FnName = IdentifierStr;
    Kind = 0;
    getNextToken();
    break;
  case tok_unary:
    getNextToken();
    if (!isascii(CurTok))
      return LogErrorP("Expected unary operator");
    FnName = "unary";
    FnName += (char)CurTok;
    Kind = 1;
    getNextToken();
    break;
  case tok_binary:
    ...
```

Giống như toán tử hai ngôi, chúng ta đặt tên cho toán tử một ngôi bằng một cái tên có chứa ký tự toán tử. Điều này giúp ích cho chúng ta lúc sinh mã. Nhân tiện nhắc đến sinh mã, mảnh cuối cùng chúng ta cần thêm là hỗ trợ sinh mã cho toán tử một ngôi. Nó trông như sau:

```c++
Value *UnaryExprAST::codegen() {
  Value *OperandV = Operand->codegen();
  if (!OperandV)
    return nullptr;

  Function *F = getFunction(std::string("unary") + Opcode);
  if (!F)
    return LogErrorV("Unknown unary operator");

  return Builder->CreateCall(F, OperandV, "unop");
}
```

Đoạn mã này tương tự, nhưng đơn giản hơn, đoạn mã cho toán tử hai ngôi. Nó đơn giản hơn chủ yếu vì không cần xử lý bất kỳ toán tử định nghĩa sẵn nào.

## Chạy thử

Thật khó tin, nhưng chỉ với vài mở rộng đơn giản trong các chương vừa qua, chúng ta đã phát triển được một ngôn ngữ gần như thực thụ. Với nó, chúng ta có thể làm rất nhiều điều thú vị, bao gồm I/O, toán học và hàng loạt thứ khác. Ví dụ, giờ đây chúng ta có thể thêm một toán tử tuần tự (sequencing) khá hay (printd được định nghĩa để in ra giá trị được chỉ định kèm một ký tự xuống dòng):

```
ready> extern printd(x);
Read extern:
declare double @printd(double)

ready> def binary : 1 (x y) 0;  # Toán tử có độ ưu tiên thấp, bỏ qua các toán hạng.
...
ready> printd(123) : printd(456) : printd(789);
123.000000
456.000000
789.000000
Evaluated to 0.000000
```

Chúng ta cũng có thể định nghĩa một loạt các phép toán "nguyên thủy" khác, chẳng hạn:

```
# Phủ định logic một ngôi (not).
def unary!(v)
  if v then
    0
  else
    1;

# Phép đổi dấu một ngôi.
def unary-(v)
  0-v;

# Định nghĩa > với cùng độ ưu tiên như <.
def binary> 10 (LHS RHS)
  RHS < LHS;

# Phép "hoặc" logic hai ngôi, không đoản mạch.
def binary| 5 (LHS RHS)
  if LHS then
    1
  else if RHS then
    1
  else
    0;

# Phép "và" logic hai ngôi, không đoản mạch.
def binary& 6 (LHS RHS)
  if !LHS then
    0
  else
    !!RHS;

# Định nghĩa = với độ ưu tiên thấp hơn một chút so với các toán tử quan hệ.
def binary = 9 (LHS RHS)
  !(LHS < RHS | LHS > RHS);

# Định nghĩa ':' để tuần tự hóa: là một toán tử có độ ưu tiên thấp, bỏ qua các toán hạng
# và chỉ trả về RHS.
def binary : 1 (x y) y;
```

Nhờ hỗ trợ if/then/else đã có, chúng ta cũng có thể định nghĩa những hàm I/O thú vị. Ví dụ, hàm sau in ra một ký tự có "mật độ" phản ánh giá trị được truyền vào: giá trị càng thấp, ký tự càng đậm đặc:

```
ready> extern putchard(char);
...
ready> def printdensity(d)
  if d > 8 then
    putchard(32)  # ' '
  else if d > 4 then
    putchard(46)  # '.'
  else if d > 2 then
    putchard(43)  # '+'
  else
    putchard(42); # '*'
...
ready> printdensity(1): printdensity(2): printdensity(3):
       printdensity(4): printdensity(5): printdensity(9):
       putchard(10);
**++.
Evaluated to 0.000000
```

Dựa trên những phép toán nguyên thủy đơn giản này, chúng ta có thể bắt đầu định nghĩa những thứ thú vị hơn. Ví dụ, đây là một hàm nhỏ xác định số lần lặp cần thiết để một hàm nhất định trên mặt phẳng phức phân kỳ:

```
# Xác định xem vị trí cụ thể có phân kỳ hay không.
# Giải z = z^2 + c trên mặt phẳng phức.
def mandelconverger(real imag iters creal cimag)
  if iters > 255 | (real*real + imag*imag > 4) then
    iters
  else
    mandelconverger(real*real - imag*imag + creal,
                    2*real*imag + cimag,
                    iters+1, creal, cimag);

# Trả về số lần lặp cần thiết để phép lặp thoát ra
def mandelconverge(real imag)
  mandelconverger(real, imag, 0, real, imag);
```

Hàm "`z = z2 + c`" này là một "sinh vật" nhỏ xinh đẹp, là nền tảng cho việc tính toán [tập Mandelbrot](http://en.wikipedia.org/wiki/Mandelbrot_set). Hàm `mandelconverge` của chúng ta trả về số lần lặp cần thiết để một quỹ đạo phức thoát ra, bão hòa ở 255. Bản thân hàm này không hữu ích lắm, nhưng nếu bạn vẽ giá trị của nó trên một mặt phẳng hai chiều, bạn sẽ thấy được tập Mandelbrot. Vì ở đây chúng ta chỉ có thể dùng putchard, đầu ra đồ họa "tuyệt vời" của chúng ta khá hạn chế, nhưng vẫn có thể nhanh chóng ghép ra thứ gì đó bằng hàm vẽ mật độ ở trên:

```
# Tính toán và vẽ tập Mandelbrot với thông tin phạm vi 2 chiều
# được chỉ định.
def mandelhelp(xmin xmax xstep   ymin ymax ystep)
  for y = ymin, y < ymax, ystep in (
    (for x = xmin, x < xmax, xstep in
       printdensity(mandelconverge(x,y)))
    : putchard(10)
  )

# mandel - Đây là hàm trợ giúp tiện lợi để vẽ tập Mandelbrot
# từ vị trí được chỉ định với độ phóng đại (Magnification) được chỉ định.
def mandel(realstart imagstart realmag imagmag)
  mandelhelp(realstart, realstart+realmag*78, realmag,
             imagstart, imagstart+imagmag*40, imagmag);
```

Với những thứ này, chúng ta có thể thử vẽ tập Mandelbrot! Hãy cùng thử xem:

```
ready> mandel(-2.3, -1.3, 0.05, 0.07);
*******************************+++++++++++*************************************
*************************+++++++++++++++++++++++*******************************
**********************+++++++++++++++++++++++++++++****************************
*******************+++++++++++++++++++++.. ...++++++++*************************
*****************++++++++++++++++++++++.... ...+++++++++***********************
***************+++++++++++++++++++++++.....   ...+++++++++*********************
**************+++++++++++++++++++++++....     ....+++++++++********************
*************++++++++++++++++++++++......      .....++++++++*******************
************+++++++++++++++++++++.......       .......+++++++******************
***********+++++++++++++++++++....                ... .+++++++*****************
**********+++++++++++++++++.......                     .+++++++****************
*********++++++++++++++...........                    ...+++++++***************
********++++++++++++............                      ...++++++++**************
********++++++++++... ..........                        .++++++++**************
*******+++++++++.....                                   .+++++++++*************
*******++++++++......                                  ..+++++++++*************
*******++++++.......                                   ..+++++++++*************
*******+++++......                                     ..+++++++++*************
*******.... ....                                      ...+++++++++*************
*******.... .                                         ...+++++++++*************
*******+++++......                                    ...+++++++++*************
*******++++++.......                                   ..+++++++++*************
*******++++++++......                                   .+++++++++*************
*******+++++++++.....                                  ..+++++++++*************
********++++++++++... ..........                        .++++++++**************
********++++++++++++............                      ...++++++++**************
*********++++++++++++++..........                     ...+++++++***************
**********++++++++++++++++........                     .+++++++****************
**********++++++++++++++++++++....                ... ..+++++++****************
***********++++++++++++++++++++++.......       .......++++++++*****************
************+++++++++++++++++++++++......      ......++++++++******************
**************+++++++++++++++++++++++....      ....++++++++********************
***************+++++++++++++++++++++++.....   ...+++++++++*********************
*****************++++++++++++++++++++++....  ...++++++++***********************
*******************+++++++++++++++++++++......++++++++*************************
*********************++++++++++++++++++++++.++++++++***************************
*************************+++++++++++++++++++++++*******************************
******************************+++++++++++++************************************
*******************************************************************************
*******************************************************************************
*******************************************************************************
Evaluated to 0.000000
ready> mandel(-2, -1, 0.02, 0.04);
**************************+++++++++++++++++++++++++++++++++++++++++++++++++++++
***********************++++++++++++++++++++++++++++++++++++++++++++++++++++++++
*********************+++++++++++++++++++++++++++++++++++++++++++++++++++++++++.
*******************+++++++++++++++++++++++++++++++++++++++++++++++++++++++++...
*****************+++++++++++++++++++++++++++++++++++++++++++++++++++++++++.....
***************++++++++++++++++++++++++++++++++++++++++++++++++++++++++........
**************++++++++++++++++++++++++++++++++++++++++++++++++++++++...........
************+++++++++++++++++++++++++++++++++++++++++++++++++++++..............
***********++++++++++++++++++++++++++++++++++++++++++++++++++........        .
**********++++++++++++++++++++++++++++++++++++++++++++++.............
********+++++++++++++++++++++++++++++++++++++++++++..................
*******+++++++++++++++++++++++++++++++++++++++.......................
******+++++++++++++++++++++++++++++++++++...........................
*****++++++++++++++++++++++++++++++++............................
*****++++++++++++++++++++++++++++...............................
****++++++++++++++++++++++++++......   .........................
***++++++++++++++++++++++++.........     ......    ...........
***++++++++++++++++++++++............
**+++++++++++++++++++++..............
**+++++++++++++++++++................
*++++++++++++++++++.................
*++++++++++++++++............ ...
*++++++++++++++..............
*+++....++++................
*..........  ...........
*
*..........  ...........
*+++....++++................
*++++++++++++++..............
*++++++++++++++++............ ...
*++++++++++++++++++.................
**+++++++++++++++++++................
**+++++++++++++++++++++..............
***++++++++++++++++++++++............
***++++++++++++++++++++++++.........     ......    ...........
****++++++++++++++++++++++++++......   .........................
*****++++++++++++++++++++++++++++...............................
*****++++++++++++++++++++++++++++++++............................
******+++++++++++++++++++++++++++++++++++...........................
*******+++++++++++++++++++++++++++++++++++++++.......................
********+++++++++++++++++++++++++++++++++++++++++++..................
Evaluated to 0.000000
ready> mandel(-0.9, -1.4, 0.02, 0.03);
*******************************************************************************
*******************************************************************************
*******************************************************************************
**********+++++++++++++++++++++************************************************
*+++++++++++++++++++++++++++++++++++++++***************************************
+++++++++++++++++++++++++++++++++++++++++++++**********************************
++++++++++++++++++++++++++++++++++++++++++++++++++*****************************
++++++++++++++++++++++++++++++++++++++++++++++++++++++*************************
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++**********************
+++++++++++++++++++++++++++++++++.........++++++++++++++++++*******************
+++++++++++++++++++++++++++++++....   ......+++++++++++++++++++****************
+++++++++++++++++++++++++++++.......  ........+++++++++++++++++++**************
++++++++++++++++++++++++++++........   ........++++++++++++++++++++************
+++++++++++++++++++++++++++.........     ..  ...+++++++++++++++++++++**********
++++++++++++++++++++++++++...........        ....++++++++++++++++++++++********
++++++++++++++++++++++++.............       .......++++++++++++++++++++++******
+++++++++++++++++++++++.............        ........+++++++++++++++++++++++****
++++++++++++++++++++++...........           ..........++++++++++++++++++++++***
++++++++++++++++++++...........                .........++++++++++++++++++++++*
++++++++++++++++++............                  ...........++++++++++++++++++++
++++++++++++++++...............                 .............++++++++++++++++++
++++++++++++++.................                 ...............++++++++++++++++
++++++++++++..................                  .................++++++++++++++
+++++++++..................                      .................+++++++++++++
++++++........        .                               .........  ..++++++++++++
++............                                         ......    ....++++++++++
..............                                                    ...++++++++++
..............                                                    ....+++++++++
..............                                                    .....++++++++
.............                                                    ......++++++++
...........                                                     .......++++++++
.........                                                       ........+++++++
.........                                                       ........+++++++
.........                                                           ....+++++++
........                                                             ...+++++++
.......                                                              ...+++++++
                                                                    ....+++++++
                                                                   .....+++++++
                                                                    ....+++++++
                                                                    ....+++++++
                                                                    ....+++++++
Evaluated to 0.000000
ready> ^D
```

Đến đây, có lẽ bạn đang bắt đầu nhận ra rằng Kaleidoscope là một ngôn ngữ thực thụ và mạnh mẽ. Nó có thể không tự đồng dạng (self-similar) :), nhưng nó có thể được dùng để vẽ những thứ tự đồng dạng!

Như vậy, chúng ta đã kết thúc chương "thêm toán tử do người dùng định nghĩa" của loạt hướng dẫn. Chúng ta đã mở rộng thành công ngôn ngữ của mình, bổ sung khả năng mở rộng ngôn ngữ ngay trong thư viện, và đã cho thấy cách tận dụng điều này để xây dựng một ứng dụng đơn giản nhưng thú vị cho người dùng cuối bằng Kaleidoscope. Hiện tại, Kaleidoscope có thể xây dựng nhiều loại ứng dụng có chức năng thực sự và có thể gọi các hàm có hiệu ứng phụ (side-effect), nhưng nó chưa thể tự định nghĩa và thay đổi giá trị của một biến.

Đáng chú ý là việc thay đổi giá trị biến (variable mutation) là một tính năng quan trọng của một số ngôn ngữ, và hoàn toàn không dễ thấy làm thế nào để [thêm hỗ trợ cho biến khả biến](LangImpl07.md) mà không phải thêm một giai đoạn "xây dựng SSA" (SSA construction) vào front-end của bạn. Trong chương tiếp theo, chúng ta sẽ mô tả cách thêm khả năng thay đổi giá trị biến mà không cần xây dựng SSA trong front-end.

## Mã nguồn đầy đủ

Dưới đây là toàn bộ mã nguồn cho ví dụ xuyên suốt của chúng ta, được bổ sung hỗ trợ cho toán tử do người dùng định nghĩa. Để biên dịch ví dụ này, hãy dùng:

```bash
# Biên dịch
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
# Chạy
./toy
```

Trên một số nền tảng, bạn sẽ cần chỉ định -rdynamic hoặc -Wl,--export-dynamic khi liên kết. Điều này đảm bảo các ký hiệu được định nghĩa trong file thực thi chính được xuất (export) ra cho trình liên kết động (dynamic linker) và nhờ đó có thể được dùng để phân giải ký hiệu lúc chạy. Việc này không cần thiết nếu bạn biên dịch mã hỗ trợ thành một thư viện chia sẻ (shared library), dù làm vậy sẽ gây ra vấn đề trên Windows.

Đây là mã nguồn:

```{literalinclude} ../../../examples/Kaleidoscope/Chapter6/toy.cpp
:language: c++
```

[Tiếp theo: Mở rộng ngôn ngữ: biến khả biến / xây dựng SSA](LangImpl07.vi.md)
