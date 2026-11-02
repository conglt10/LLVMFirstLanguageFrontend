# Kaleidoscope: Mở rộng ngôn ngữ: Luồng điều khiển

## Giới thiệu Chương 5

Chào mừng bạn đến với Chương 5 của loạt hướng dẫn "[Hiện thực một ngôn ngữ với LLVM](index.md)". Các phần 1-4 đã mô tả việc hiện thực ngôn ngữ Kaleidoscope đơn giản, bao gồm hỗ trợ sinh LLVM IR, tiếp theo là các tối ưu hóa và một trình biên dịch JIT. Đáng tiếc là, ở dạng hiện tại, Kaleidoscope gần như vô dụng: nó không có luồng điều khiển nào ngoài lời gọi hàm và lệnh trả về. Điều này có nghĩa là bạn không thể có các nhánh rẽ có điều kiện trong mã, làm hạn chế đáng kể sức mạnh của ngôn ngữ. Trong tập này của loạt bài "xây dựng trình biên dịch", chúng ta sẽ mở rộng Kaleidoscope để có biểu thức if/then/else cùng với một vòng lặp 'for' đơn giản.

## If/Then/Else

Việc mở rộng Kaleidoscope để hỗ trợ if/then/else khá đơn giản. Về cơ bản, nó yêu cầu bổ sung hỗ trợ cho khái niệm "mới" này vào lexer (bộ phân tích từ vựng), parser (bộ phân tích cú pháp), AST và phần phát sinh mã LLVM. Ví dụ này rất hay, vì nó cho thấy việc "phát triển" một ngôn ngữ theo thời gian dễ dàng như thế nào, mở rộng nó từng bước khi phát hiện ra những ý tưởng mới.

Trước khi bắt tay vào việc thêm phần mở rộng này "như thế nào", hãy nói về việc chúng ta muốn "cái gì". Ý tưởng cơ bản là chúng ta muốn có thể viết những thứ như sau:

```
def fib(x)
  if x < 3 then
    1
  else
    fib(x-1)+fib(x-2);
```

Trong Kaleidoscope, mọi cấu trúc đều là biểu thức: không có câu lệnh (statement). Do đó, biểu thức if/then/else cần trả về một giá trị như mọi biểu thức khác. Vì chúng ta chủ yếu sử dụng dạng hàm (functional), chúng ta sẽ cho nó đánh giá điều kiện, rồi trả về giá trị 'then' hoặc 'else' tùy theo kết quả của điều kiện. Điều này rất giống với biểu thức "?:" trong C.

Ngữ nghĩa của biểu thức if/then/else là nó đánh giá điều kiện thành một giá trị boolean: 0.0 được coi là false và mọi giá trị khác được coi là true. Nếu điều kiện là true, biểu thức con thứ nhất được đánh giá và trả về; nếu điều kiện là false, biểu thức con thứ hai được đánh giá và trả về. Vì Kaleidoscope cho phép hiệu ứng phụ (side-effect), việc xác định rõ hành vi này là rất quan trọng.

Giờ khi đã biết mình "muốn" gì, hãy chia nhỏ nó thành các thành phần cấu tạo.

### Mở rộng Lexer cho If/Then/Else

Các mở rộng cho lexer rất đơn giản. Đầu tiên, chúng ta thêm các giá trị enum mới cho các token tương ứng:

```c++
// điều khiển
tok_if = -6,
tok_then = -7,
tok_else = -8,
```

Sau đó, chúng ta nhận diện các từ khóa mới trong lexer. Việc này khá đơn giản:

```c++
...
if (IdentifierStr == "def")
  return tok_def;
if (IdentifierStr == "extern")
  return tok_extern;
if (IdentifierStr == "if")
  return tok_if;
if (IdentifierStr == "then")
  return tok_then;
if (IdentifierStr == "else")
  return tok_else;
return tok_identifier;
```

### Mở rộng AST cho If/Then/Else

Để biểu diễn biểu thức mới, chúng ta thêm một nút AST mới cho nó:

```c++
/// IfExprAST - Lớp biểu thức cho if/then/else.
class IfExprAST : public ExprAST {
  std::unique_ptr<ExprAST> Cond, Then, Else;

public:
  IfExprAST(std::unique_ptr<ExprAST> Cond, std::unique_ptr<ExprAST> Then,
            std::unique_ptr<ExprAST> Else)
    : Cond(std::move(Cond)), Then(std::move(Then)), Else(std::move(Else)) {}

  Value *codegen() override;
};
```

Nút AST này chỉ chứa các con trỏ đến những biểu thức con khác nhau.

### Mở rộng Parser cho If/Then/Else

Giờ đây khi đã có các token liên quan từ lexer và có nút AST để xây dựng, logic phân tích cú pháp của chúng ta tương đối đơn giản. Đầu tiên, chúng ta định nghĩa một hàm phân tích mới:

```c++
/// ifexpr ::= 'if' expression 'then' expression 'else' expression
static std::unique_ptr<ExprAST> ParseIfExpr() {
  getNextToken();  // tiêu thụ token if.

  // điều kiện.
  auto Cond = ParseExpression();
  if (!Cond)
    return nullptr;

  if (CurTok != tok_then)
    return LogError("expected then");
  getNextToken();  // tiêu thụ token then

  auto Then = ParseExpression();
  if (!Then)
    return nullptr;

  if (CurTok != tok_else)
    return LogError("expected else");

  getNextToken();

  auto Else = ParseExpression();
  if (!Else)
    return nullptr;

  return std::make_unique<IfExprAST>(std::move(Cond), std::move(Then),
                                      std::move(Else));
}
```

Tiếp theo, chúng ta kết nối nó như một biểu thức cơ sở (primary expression):

```c++
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
  }
}
```

### LLVM IR cho If/Then/Else

Giờ đây khi đã có phần phân tích cú pháp và xây dựng AST, mảnh ghép cuối cùng là bổ sung hỗ trợ sinh mã LLVM. Đây là phần thú vị nhất của ví dụ if/then/else, vì đây là nơi bắt đầu xuất hiện các khái niệm mới. Toàn bộ phần mã ở trên đã được mô tả kỹ lưỡng trong các chương trước.

Để hiểu rõ mã mà chúng ta muốn sinh ra, hãy xem một ví dụ đơn giản. Xét:

```
extern foo();
extern bar();
def baz(x) if x then foo() else bar();
```

Nếu bạn tắt các tối ưu hóa, mã mà bạn (sắp) nhận được từ Kaleidoscope sẽ trông như sau:

```llvm
declare double @foo()

declare double @bar()

define double @baz(double %x) {
entry:
  %ifcond = fcmp one double %x, 0.000000e+00
  br i1 %ifcond, label %then, label %else

then:       ; preds = %entry
  %calltmp = call double @foo()
  br label %ifcont

else:       ; preds = %entry
  %calltmp1 = call double @bar()
  br label %ifcont

ifcont:     ; preds = %else, %then
  %iftmp = phi double [ %calltmp, %then ], [ %calltmp1, %else ]
  ret double %iftmp
}
```

Để trực quan hóa đồ thị luồng điều khiển (control flow graph - CFG), bạn có thể dùng một tính năng tiện lợi của công cụ '[opt](https://llvm.org/cmds/opt.html)' trong LLVM. Nếu bạn đặt LLVM IR này vào "t.ll" và chạy "`llvm-as < t.ll | opt -passes=view-cfg`", [một cửa sổ sẽ hiện lên](../../ProgrammersManual.md#viewing-graphs-while-debugging-code) và bạn sẽ thấy đồ thị này:

:::{figure} LangImpl05-cfg.png
:align: center
:alt: CFG ví dụ

CFG ví dụ
:::

Một cách khác để có được đồ thị này là gọi "`F->viewCFG()`" hoặc "`F->viewCFGOnly()`" (trong đó F là một "`Function*`"), bằng cách chèn các lời gọi thực sự vào mã rồi biên dịch lại, hoặc bằng cách gọi chúng trong trình gỡ lỗi (debugger). LLVM có nhiều tính năng hay để trực quan hóa các loại đồ thị khác nhau.

Quay lại với mã được sinh ra, nó khá đơn giản: khối entry đánh giá biểu thức điều kiện ("x" trong trường hợp này) và so sánh kết quả với 0.0 bằng lệnh "`fcmp one`" ('one' nghĩa là "Ordered and Not Equal" – có thứ tự và không bằng). Dựa trên kết quả của biểu thức này, mã sẽ nhảy đến khối "then" hoặc "else", nơi chứa các biểu thức cho trường hợp true/false.

Khi các khối then/else thực thi xong, cả hai đều rẽ nhánh quay về khối 'ifcont' để thực thi phần mã nằm sau if/then/else. Trong trường hợp này, việc duy nhất còn lại là trả về cho nơi gọi hàm. Câu hỏi đặt ra là: làm sao mã biết được nên trả về biểu thức nào?

Câu trả lời cho câu hỏi này liên quan đến một phép toán SSA quan trọng: [phép toán Phi](http://en.wikipedia.org/wiki/Static_single_assignment_form). Nếu bạn chưa quen với SSA, [bài viết trên wikipedia](http://en.wikipedia.org/wiki/Static_single_assignment_form) là một phần giới thiệu tốt, và còn nhiều bài giới thiệu khác mà bạn có thể tìm thấy qua công cụ tìm kiếm yêu thích của mình. Nói ngắn gọn, việc "thực thi" phép toán Phi đòi hỏi phải "nhớ" luồng điều khiển đến từ khối nào. Phép toán Phi nhận giá trị tương ứng với khối điều khiển đầu vào. Trong trường hợp này, nếu luồng điều khiển đến từ khối "then", nó nhận giá trị của "calltmp". Nếu luồng điều khiển đến từ khối "else", nó nhận giá trị của "calltmp1".

Đến đây, có lẽ bạn bắt đầu nghĩ "Ôi không! Điều này có nghĩa là front-end đơn giản và thanh lịch của tôi sẽ phải bắt đầu sinh dạng SSA để có thể dùng LLVM!". May mắn là không phải vậy, và chúng tôi đặc biệt khuyên bạn *không nên* hiện thực thuật toán xây dựng SSA trong front-end của mình trừ khi có một lý do cực kỳ chính đáng. Trong thực tế, có hai loại giá trị xuất hiện trong mã viết cho một ngôn ngữ lập trình mệnh lệnh thông thường có thể cần đến nút Phi:

1. Mã liên quan đến biến của người dùng: `x = 1; x = x + 1;`
2. Các giá trị ngầm định trong cấu trúc AST của bạn, chẳng hạn như nút Phi trong trường hợp này.

Trong [Chương 7](LangImpl07.md) của hướng dẫn này ("biến có thể thay đổi"), chúng ta sẽ bàn sâu về trường hợp #1. Hiện tại, bạn cứ tin tôi rằng bạn không cần xây dựng SSA để xử lý trường hợp này. Với #2, bạn có thể chọn sử dụng các kỹ thuật mà chúng ta sẽ mô tả cho #1, hoặc có thể chèn trực tiếp các nút Phi nếu thuận tiện. Trong trường hợp này, việc sinh nút Phi thực sự rất dễ, nên chúng ta chọn làm trực tiếp.

Được rồi, đủ phần động lực và tổng quan rồi, hãy sinh mã thôi!

### Sinh mã cho If/Then/Else

Để sinh mã cho phần này, chúng ta hiện thực phương thức `codegen` cho `IfExprAST`:

```c++
Value *IfExprAST::codegen() {
  Value *CondV = Cond->codegen();
  if (!CondV)
    return nullptr;

  // Chuyển điều kiện thành bool bằng cách so sánh khác với 0.0.
  CondV = Builder->CreateFCmpONE(
      CondV, ConstantFP::get(*TheContext, APFloat(0.0)), "ifcond");
```

Đoạn mã này đơn giản và tương tự những gì chúng ta đã thấy trước đây. Chúng ta phát sinh (emit) biểu thức cho điều kiện, rồi so sánh giá trị đó với 0 để nhận được giá trị chân lý dưới dạng giá trị 1 bit (bool).

```c++
Function *TheFunction = Builder->GetInsertBlock()->getParent();

// Tạo các khối cho trường hợp then và else. Chèn khối 'then' vào
// cuối hàm.
BasicBlock *ThenBB =
    BasicBlock::Create(*TheContext, "then", TheFunction);
BasicBlock *ElseBB = BasicBlock::Create(*TheContext, "else");
BasicBlock *MergeBB = BasicBlock::Create(*TheContext, "ifcont");

Builder->CreateCondBr(CondV, ThenBB, ElseBB);
```

Đoạn mã này tạo các khối cơ bản (basic block) liên quan đến câu lệnh if/then/else, và tương ứng trực tiếp với các khối trong ví dụ ở trên. Dòng đầu tiên lấy đối tượng Function hiện tại đang được xây dựng. Nó làm điều này bằng cách hỏi builder về BasicBlock hiện tại, rồi hỏi khối đó về "parent" (cha) của nó (tức hàm mà khối đang được nhúng vào).

Sau khi có được hàm, nó tạo ba khối. Lưu ý rằng nó truyền "TheFunction" vào constructor của khối "then". Điều này khiến constructor tự động chèn khối mới vào cuối hàm được chỉ định. Hai khối còn lại được tạo ra nhưng chưa được chèn vào hàm.

Khi các khối đã được tạo, chúng ta có thể phát sinh lệnh rẽ nhánh có điều kiện để chọn giữa chúng. Lưu ý rằng việc tạo khối mới không ngầm ảnh hưởng đến IRBuilder, nên nó vẫn đang chèn vào khối chứa điều kiện. Cũng lưu ý rằng nó đang tạo một nhánh đến khối "then" và khối "else", mặc dù khối "else" chưa được chèn vào hàm. Điều này hoàn toàn ổn: đó là cách tiêu chuẩn mà LLVM hỗ trợ tham chiếu tới trước (forward reference).

```c++
// Phát sinh giá trị then.
Builder->SetInsertPoint(ThenBB);

Value *ThenV = Then->codegen();
if (!ThenV)
  return nullptr;

Builder->CreateBr(MergeBB);
// Codegen của 'Then' có thể thay đổi khối hiện tại, cập nhật ThenBB cho PHI.
ThenBB = Builder->GetInsertBlock();
```

Sau khi lệnh rẽ nhánh có điều kiện được chèn, chúng ta di chuyển builder để bắt đầu chèn vào khối "then". Nói một cách chính xác, lời gọi này di chuyển điểm chèn đến cuối khối được chỉ định. Tuy nhiên, vì khối "then" đang rỗng, nên nó cũng đồng thời bắt đầu chèn từ đầu khối. :)

Khi điểm chèn đã được đặt, chúng ta gọi codegen đệ quy cho biểu thức "then" từ AST. Để hoàn tất khối "then", chúng ta tạo một lệnh rẽ nhánh không điều kiện đến khối hợp nhất (merge block). Một khía cạnh thú vị (và rất quan trọng) của LLVM IR là nó {ref}`yêu cầu mọi khối cơ bản phải được "kết thúc" <functionstructure>` bằng một {ref}`lệnh luồng điều khiển <terminators>` như return hoặc branch. Điều này có nghĩa là mọi luồng điều khiển, *kể cả fall through (chảy tuần tự xuống khối kế tiếp)*, đều phải được thể hiện tường minh trong LLVM IR. Nếu bạn vi phạm quy tắc này, bộ kiểm tra (verifier) sẽ báo lỗi.

Dòng cuối cùng ở đây khá tinh tế nhưng rất quan trọng. Vấn đề cơ bản là khi tạo nút Phi trong khối hợp nhất, chúng ta cần thiết lập các cặp khối/giá trị chỉ ra cách Phi sẽ hoạt động. Điều quan trọng là nút Phi cần có một mục cho mỗi khối tiền nhiệm (predecessor) của khối đó trong CFG. Vậy tại sao chúng ta lại lấy khối hiện tại trong khi vừa đặt nó thành ThenBB ở 5 dòng phía trên? Vấn đề là bản thân biểu thức "Then" có thể thực sự thay đổi khối mà Builder đang phát sinh mã vào, chẳng hạn nếu nó chứa một biểu thức "if/then/else" lồng nhau. Vì việc gọi `codegen()` đệ quy có thể thay đổi tùy ý khối hiện tại, chúng ta buộc phải lấy giá trị mới nhất cho phần mã sẽ thiết lập nút Phi.

```c++
// Phát sinh khối else.
TheFunction->insert(TheFunction->end(), ElseBB);
Builder->SetInsertPoint(ElseBB);

Value *ElseV = Else->codegen();
if (!ElseV)
  return nullptr;

Builder->CreateBr(MergeBB);
// Codegen của 'Else' có thể thay đổi khối hiện tại, cập nhật ElseBB cho PHI.
ElseBB = Builder->GetInsertBlock();
```

Việc sinh mã cho khối 'else' về cơ bản giống hệt với khối 'then'. Điểm khác biệt đáng kể duy nhất là dòng đầu tiên, dòng này thêm khối 'else' vào hàm. Hãy nhớ lại rằng trước đó khối 'else' đã được tạo nhưng chưa được thêm vào hàm. Giờ khi các khối 'then' và 'else' đã được phát sinh, chúng ta có thể hoàn tất với phần mã hợp nhất:

```c++
  // Phát sinh khối hợp nhất.
  TheFunction->insert(TheFunction->end(), MergeBB);
  Builder->SetInsertPoint(MergeBB);
  PHINode *PN =
    Builder->CreatePHI(Type::getDoubleTy(*TheContext), 2, "iftmp");

  PN->addIncoming(ThenV, ThenBB);
  PN->addIncoming(ElseV, ElseBB);
  return PN;
}
```

Hai dòng đầu tiên ở đây giờ đã quen thuộc: dòng thứ nhất thêm khối "merge" vào đối tượng Function (trước đó nó ở trạng thái lơ lửng, giống như khối else ở trên). Dòng thứ hai thay đổi điểm chèn để mã mới được tạo sẽ đi vào khối "merge". Sau đó, chúng ta cần tạo nút PHI và thiết lập các cặp khối/giá trị cho PHI.

Cuối cùng, hàm CodeGen trả về nút phi làm giá trị được tính bởi biểu thức if/then/else. Trong ví dụ ở trên, giá trị trả về này sẽ được đưa vào mã của hàm cấp cao nhất (top-level function), nơi sẽ tạo lệnh return.

Nhìn chung, giờ đây chúng ta đã có khả năng thực thi mã có điều kiện trong Kaleidoscope. Với phần mở rộng này, Kaleidoscope đã là một ngôn ngữ khá hoàn chỉnh, có thể tính toán nhiều loại hàm số khác nhau. Tiếp theo, chúng ta sẽ thêm một biểu thức hữu ích khác vốn quen thuộc trong các ngôn ngữ phi hàm (non-functional)...

## Biểu thức vòng lặp 'for'

Giờ đây khi đã biết cách thêm các cấu trúc luồng điều khiển cơ bản vào ngôn ngữ, chúng ta đã có công cụ để thêm những thứ mạnh mẽ hơn. Hãy thêm một thứ tham vọng hơn, đó là biểu thức 'for':

```
extern putchard(char);
def printstar(n)
  for i = 1, i < n, 1.0 in
    putchard(42);  # ascii 42 = '*'

# in ra 100 ký tự '*'
printstar(100);
```

Biểu thức này định nghĩa một biến mới ("i" trong trường hợp này), biến này lặp từ một giá trị bắt đầu, trong khi điều kiện ("i < n" trong trường hợp này) còn đúng, và tăng dần theo một giá trị bước nhảy tùy chọn ("1.0" trong trường hợp này). Nếu bỏ qua giá trị bước nhảy, nó mặc định là 1.0. Trong khi điều kiện vòng lặp còn đúng, nó thực thi biểu thức thân vòng lặp. Vì không có gì tốt hơn để trả về, chúng ta sẽ định nghĩa vòng lặp luôn trả về 0.0. Trong tương lai, khi đã có biến có thể thay đổi, nó sẽ trở nên hữu ích hơn.

Như trước, hãy nói về những thay đổi cần thiết cho Kaleidoscope để hỗ trợ tính năng này.

### Mở rộng Lexer cho vòng lặp 'for'

Các mở rộng cho lexer cũng tương tự như với if/then/else:

```c++
... trong enum Token ...
// điều khiển
tok_if = -6, tok_then = -7, tok_else = -8,
tok_for = -9, tok_in = -10

... trong gettok ...
if (IdentifierStr == "def")
  return tok_def;
if (IdentifierStr == "extern")
  return tok_extern;
if (IdentifierStr == "if")
  return tok_if;
if (IdentifierStr == "then")
  return tok_then;
if (IdentifierStr == "else")
  return tok_else;
if (IdentifierStr == "for")
  return tok_for;
if (IdentifierStr == "in")
  return tok_in;
return tok_identifier;
```

### Mở rộng AST cho vòng lặp 'for'

Nút AST cũng đơn giản như vậy. Về cơ bản, nó chỉ cần lưu tên biến và các biểu thức thành phần trong nút.

```c++
/// ForExprAST - Lớp biểu thức cho for/in.
class ForExprAST : public ExprAST {
  std::string VarName;
  std::unique_ptr<ExprAST> Start, End, Step, Body;

public:
  ForExprAST(const std::string &VarName, std::unique_ptr<ExprAST> Start,
             std::unique_ptr<ExprAST> End, std::unique_ptr<ExprAST> Step,
             std::unique_ptr<ExprAST> Body)
    : VarName(VarName), Start(std::move(Start)), End(std::move(End)),
      Step(std::move(Step)), Body(std::move(Body)) {}

  Value *codegen() override;
};
```

### Mở rộng Parser cho vòng lặp 'for'

Mã parser cũng khá tiêu chuẩn. Điều thú vị duy nhất ở đây là việc xử lý giá trị bước nhảy tùy chọn. Mã parser xử lý nó bằng cách kiểm tra xem có dấu phẩy thứ hai hay không. Nếu không, nó đặt giá trị bước nhảy là null trong nút AST:

```c++
/// forexpr ::= 'for' identifier '=' expr ',' expr (',' expr)? 'in' expression
static std::unique_ptr<ExprAST> ParseForExpr() {
  getNextToken();  // tiêu thụ token for.

  if (CurTok != tok_identifier)
    return LogError("expected identifier after for");

  std::string IdName = IdentifierStr;
  getNextToken();  // tiêu thụ định danh.

  if (CurTok != '=')
    return LogError("expected '=' after for");
  getNextToken();  // tiêu thụ '='.


  auto Start = ParseExpression();
  if (!Start)
    return nullptr;
  if (CurTok != ',')
    return LogError("expected ',' after for start value");
  getNextToken();

  auto End = ParseExpression();
  if (!End)
    return nullptr;

  // Giá trị bước nhảy là tùy chọn.
  std::unique_ptr<ExprAST> Step;
  if (CurTok == ',') {
    getNextToken();
    Step = ParseExpression();
    if (!Step)
      return nullptr;
  }

  if (CurTok != tok_in)
    return LogError("expected 'in' after for");
  getNextToken();  // tiêu thụ 'in'.

  auto Body = ParseExpression();
  if (!Body)
    return nullptr;

  return std::make_unique<ForExprAST>(IdName, std::move(Start),
                                       std::move(End), std::move(Step),
                                       std::move(Body));
}
```

Và một lần nữa, chúng ta kết nối nó như một biểu thức cơ sở:

```c++
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
  }
}
```

### LLVM IR cho vòng lặp 'for'

Giờ chúng ta đến phần hay nhất: LLVM IR mà chúng ta muốn sinh cho cấu trúc này. Với ví dụ đơn giản ở trên, chúng ta nhận được LLVM IR sau (lưu ý rằng bản dump này được sinh khi tắt các tối ưu hóa để cho rõ ràng):

```llvm
declare double @putchard(double)

define double @printstar(double %n) {
entry:
  ; giá trị khởi đầu = 1.0 (được inline vào phi)
  br label %loop

loop:       ; preds = %loop, %entry
  %i = phi double [ 1.000000e+00, %entry ], [ %nextvar, %loop ]
  ; thân vòng lặp
  %calltmp = call double @putchard(double 4.200000e+01)
  ; bước tăng
  %nextvar = fadd double %i, 1.000000e+00

  ; kiểm tra điều kiện kết thúc
  %cmptmp = fcmp ult double %i, %n
  %booltmp = uitofp i1 %cmptmp to double
  %loopcond = fcmp one double %booltmp, 0.000000e+00
  br i1 %loopcond, label %loop, label %afterloop

afterloop:      ; preds = %loop
  ; vòng lặp luôn trả về 0.0
  ret double 0.000000e+00
}
```

Vòng lặp này chứa tất cả các cấu trúc mà chúng ta đã thấy trước đây: một nút phi, một vài biểu thức và một số khối cơ bản. Hãy xem chúng kết hợp với nhau như thế nào.

### Sinh mã cho vòng lặp 'for'

Phần đầu tiên của codegen rất đơn giản: chúng ta chỉ cần xuất ra biểu thức bắt đầu cho giá trị vòng lặp:

```c++
Value *ForExprAST::codegen() {
  // Phát sinh mã bắt đầu trước, khi 'biến' chưa nằm trong phạm vi.
  Value *StartVal = Start->codegen();
  if (!StartVal)
    return nullptr;
```

Xong phần đó, bước tiếp theo là thiết lập khối cơ bản LLVM cho phần bắt đầu của thân vòng lặp. Trong trường hợp ở trên, toàn bộ thân vòng lặp là một khối, nhưng hãy nhớ rằng bản thân mã của thân vòng lặp có thể gồm nhiều khối (ví dụ: nếu nó chứa một biểu thức if/then/else hoặc for/in).

```c++
// Tạo khối cơ bản mới cho phần đầu vòng lặp, chèn sau khối
// hiện tại.
Function *TheFunction = Builder->GetInsertBlock()->getParent();
BasicBlock *PreheaderBB = Builder->GetInsertBlock();
BasicBlock *LoopBB =
    BasicBlock::Create(*TheContext, "loop", TheFunction);

// Chèn một lệnh fall through tường minh từ khối hiện tại đến LoopBB.
Builder->CreateBr(LoopBB);
```

Đoạn mã này tương tự những gì chúng ta đã thấy với if/then/else. Vì sẽ cần đến nó để tạo nút Phi, chúng ta ghi nhớ khối chảy (fall through) vào vòng lặp. Sau đó, chúng ta tạo khối thực sự bắt đầu vòng lặp và tạo một lệnh rẽ nhánh không điều kiện cho việc fall-through giữa hai khối.

```c++
// Bắt đầu chèn vào LoopBB.
Builder->SetInsertPoint(LoopBB);

// Khởi tạo nút PHI với một mục cho Start.
PHINode *Variable = Builder->CreatePHI(Type::getDoubleTy(*TheContext),
                                       2, VarName);
Variable->addIncoming(StartVal, PreheaderBB);
```

Giờ khi "preheader" (khối tiền đầu) của vòng lặp đã được thiết lập, chúng ta chuyển sang phát sinh mã cho thân vòng lặp. Đầu tiên, chúng ta di chuyển điểm chèn và tạo nút PHI cho biến quy nạp (induction variable) của vòng lặp. Vì đã biết giá trị đầu vào cho giá trị khởi đầu, chúng ta thêm nó vào nút Phi. Lưu ý rằng cuối cùng Phi sẽ nhận một giá trị thứ hai cho cạnh quay lui (backedge), nhưng chúng ta chưa thể thiết lập nó lúc này (vì nó chưa tồn tại!).

```c++
// Bên trong vòng lặp, biến được định nghĩa bằng nút PHI. Nếu nó
// che khuất một biến đã tồn tại, ta phải khôi phục biến đó, nên lưu lại ngay bây giờ.
Value *OldVal = NamedValues[VarName];
NamedValues[VarName] = Variable;

// Phát sinh thân vòng lặp. Cũng như mọi biểu thức khác, nó có thể thay đổi
// BB hiện tại. Lưu ý rằng ta bỏ qua giá trị được tính bởi thân vòng lặp, nhưng
// không cho phép lỗi.
if (!Body->codegen())
  return nullptr;
```

Giờ mã bắt đầu trở nên thú vị hơn. Vòng lặp 'for' của chúng ta đưa một biến mới vào bảng ký hiệu (symbol table). Điều này có nghĩa là bảng ký hiệu giờ có thể chứa tham số hàm hoặc biến vòng lặp. Để xử lý điều này, trước khi sinh mã cho thân vòng lặp, chúng ta thêm biến vòng lặp làm giá trị hiện tại cho tên của nó. Lưu ý rằng có thể tồn tại một biến cùng tên ở phạm vi bên ngoài. Việc biến điều này thành lỗi rất dễ (báo lỗi và trả về null nếu đã có một mục cho VarName), nhưng chúng ta chọn cho phép che khuất (shadowing) biến. Để xử lý đúng điều này, chúng ta ghi nhớ Value có thể đang bị che khuất vào `OldVal` (sẽ là null nếu không có biến nào bị che khuất).

Khi biến vòng lặp đã được đặt vào bảng ký hiệu, mã sẽ gọi codegen đệ quy cho thân vòng lặp. Điều này cho phép thân vòng lặp sử dụng biến vòng lặp: mọi tham chiếu đến nó sẽ tự nhiên tìm thấy nó trong bảng ký hiệu.

```c++
// Phát sinh giá trị bước nhảy.
Value *StepVal = nullptr;
if (Step) {
  StepVal = Step->codegen();
  if (!StepVal)
    return nullptr;
} else {
  // Nếu không được chỉ định, dùng 1.0.
  StepVal = ConstantFP::get(*TheContext, APFloat(1.0));
}

Value *NextVar = Builder->CreateFAdd(Variable, StepVal, "nextvar");
```

Giờ khi thân vòng lặp đã được phát sinh, chúng ta tính giá trị tiếp theo của biến lặp bằng cách cộng thêm giá trị bước nhảy, hoặc 1.0 nếu không có. '`NextVar`' sẽ là giá trị của biến vòng lặp ở lần lặp tiếp theo.

```c++
// Tính điều kiện kết thúc.
Value *EndCond = End->codegen();
if (!EndCond)
  return nullptr;

// Chuyển điều kiện thành bool bằng cách so sánh khác với 0.0.
EndCond = Builder->CreateFCmpONE(
    EndCond, ConstantFP::get(*TheContext, APFloat(0.0)), "loopcond");
```

Cuối cùng, chúng ta đánh giá giá trị thoát của vòng lặp để xác định xem vòng lặp có nên kết thúc hay không. Phần này tương tự việc đánh giá điều kiện của câu lệnh if/then/else.

```c++
// Tạo khối "after loop" và chèn nó vào.
BasicBlock *LoopEndBB = Builder->GetInsertBlock();
BasicBlock *AfterBB =
    BasicBlock::Create(*TheContext, "afterloop", TheFunction);

// Chèn lệnh rẽ nhánh có điều kiện vào cuối LoopEndBB.
Builder->CreateCondBr(EndCond, LoopBB, AfterBB);

// Mọi mã mới sẽ được chèn vào AfterBB.
Builder->SetInsertPoint(AfterBB);
```

Khi mã cho thân vòng lặp đã hoàn chỉnh, chúng ta chỉ cần hoàn tất luồng điều khiển cho nó. Đoạn mã này ghi nhớ khối kết thúc (cho nút phi), sau đó tạo khối cho lối thoát vòng lặp ("afterloop"). Dựa trên giá trị của điều kiện thoát, nó tạo một lệnh rẽ nhánh có điều kiện để chọn giữa việc thực thi vòng lặp thêm lần nữa và thoát khỏi vòng lặp. Mọi mã phía sau đều được phát sinh trong khối "afterloop", vì vậy nó đặt vị trí chèn vào khối đó.

```c++
  // Thêm một mục mới vào nút PHI cho cạnh quay lui.
  Variable->addIncoming(NextVar, LoopEndBB);

  // Khôi phục biến đã bị che khuất.
  if (OldVal)
    NamedValues[VarName] = OldVal;
  else
    NamedValues.erase(VarName);

  // Biểu thức for luôn trả về 0.0.
  return Constant::getNullValue(Type::getDoubleTy(*TheContext));
}
```

Phần mã cuối cùng xử lý các công việc dọn dẹp: giờ khi đã có giá trị "NextVar", chúng ta có thể thêm giá trị đầu vào này vào nút PHI của vòng lặp. Sau đó, chúng ta xóa biến vòng lặp khỏi bảng ký hiệu, để nó không còn nằm trong phạm vi sau vòng lặp for. Cuối cùng, việc sinh mã cho vòng lặp for luôn trả về 0.0, nên đó là giá trị chúng ta trả về từ `ForExprAST::codegen()`.

Đến đây, chúng ta kết thúc chương "thêm luồng điều khiển vào Kaleidoscope" của hướng dẫn. Trong chương này, chúng ta đã thêm hai cấu trúc luồng điều khiển và dùng chúng để làm rõ một vài khía cạnh của LLVM IR mà những người hiện thực front-end cần biết. Trong chương tiếp theo của câu chuyện, chúng ta sẽ "điên rồ" hơn một chút và thêm [toán tử do người dùng định nghĩa](LangImpl06.md) vào ngôn ngữ nhỏ bé ngây thơ của chúng ta.

## Mã nguồn đầy đủ

Dưới đây là mã nguồn đầy đủ của ví dụ xuyên suốt, được bổ sung thêm các biểu thức if/then/else và for. Để build ví dụ này, hãy dùng:

```bash
# Biên dịch
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
# Chạy
./toy
```

Và đây là mã nguồn:

```{literalinclude} ../../../examples/Kaleidoscope/Chapter5/toy.cpp
:language: c++
```

[Tiếp theo: Mở rộng ngôn ngữ: toán tử do người dùng định nghĩa](LangImpl06.md)
