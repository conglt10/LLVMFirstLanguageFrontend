# Kaleidoscope: Hiện thực Parser và AST

## Giới thiệu Chương 2

Chào mừng bạn đến với Chương 2 của loạt hướng dẫn "[Hiện thực một ngôn ngữ với LLVM](index.md)". Chương này hướng dẫn bạn cách sử dụng lexer (bộ phân tích từ vựng) đã xây dựng ở [Chương 1](LangImpl01.md) để xây dựng một [parser](http://en.wikipedia.org/wiki/Parsing) (bộ phân tích cú pháp) hoàn chỉnh cho ngôn ngữ Kaleidoscope của chúng ta. Khi đã có parser, chúng ta sẽ định nghĩa và xây dựng một [Cây cú pháp trừu tượng](http://en.wikipedia.org/wiki/Abstract_syntax_tree) (Abstract Syntax Tree – AST).

Parser mà chúng ta sẽ xây dựng kết hợp [Phân tích đệ quy đi xuống](http://en.wikipedia.org/wiki/Recursive_descent_parser) (Recursive Descent Parsing) và [Phân tích theo độ ưu tiên toán tử](http://en.wikipedia.org/wiki/Operator-precedence_parser) (Operator-Precedence Parsing) để phân tích ngôn ngữ Kaleidoscope (phương pháp sau dùng cho biểu thức nhị phân, phương pháp trước dùng cho mọi thứ còn lại). Tuy nhiên, trước khi bắt tay vào phân tích cú pháp, hãy nói về đầu ra của parser: Cây cú pháp trừu tượng.

## Cây cú pháp trừu tượng (AST)

AST của một chương trình nắm bắt hành vi của chương trình đó theo cách giúp các giai đoạn sau của trình biên dịch (ví dụ: sinh mã) dễ dàng diễn giải. Về cơ bản, chúng ta muốn có một đối tượng cho mỗi cấu trúc trong ngôn ngữ, và AST nên mô phỏng sát ngôn ngữ. Trong Kaleidoscope, chúng ta có biểu thức, prototype (nguyên mẫu hàm) và đối tượng hàm. Hãy bắt đầu với biểu thức trước:

```c++
/// ExprAST - Lớp cơ sở cho tất cả các nút biểu thức.
class ExprAST {
public:
  virtual ~ExprAST() = default;
};

/// NumberExprAST - Lớp biểu thức cho các literal số như "1.0".
class NumberExprAST : public ExprAST {
  double Val;

public:
  NumberExprAST(double Val) : Val(Val) {}
};
```

Đoạn mã trên cho thấy định nghĩa của lớp cơ sở ExprAST và một lớp con mà chúng ta dùng cho các literal số. Điều quan trọng cần lưu ý ở đoạn mã này là lớp NumberExprAST lưu giá trị số của literal dưới dạng biến thể hiện (instance variable). Điều này cho phép các giai đoạn sau của trình biên dịch biết giá trị số được lưu trữ là gì.

Hiện tại chúng ta mới chỉ tạo AST, nên chưa có phương thức truy xuất (accessor) hữu ích nào trên chúng. Chẳng hạn, việc thêm một phương thức ảo để in mã ra một cách đẹp mắt (pretty print) là rất dễ dàng. Dưới đây là định nghĩa của các nút AST biểu thức khác mà chúng ta sẽ dùng trong dạng cơ bản của ngôn ngữ Kaleidoscope:

```c++
/// VariableExprAST - Lớp biểu thức để tham chiếu đến một biến, như "a".
class VariableExprAST : public ExprAST {
  std::string Name;

public:
  VariableExprAST(const std::string &Name) : Name(Name) {}
};

/// BinaryExprAST - Lớp biểu thức cho toán tử nhị phân.
class BinaryExprAST : public ExprAST {
  char Op;
  std::unique_ptr<ExprAST> LHS, RHS;

public:
  BinaryExprAST(char Op, std::unique_ptr<ExprAST> LHS,
                std::unique_ptr<ExprAST> RHS)
    : Op(Op), LHS(std::move(LHS)), RHS(std::move(RHS)) {}
};

/// CallExprAST - Lớp biểu thức cho lời gọi hàm.
class CallExprAST : public ExprAST {
  std::string Callee;
  std::vector<std::unique_ptr<ExprAST>> Args;

public:
  CallExprAST(const std::string &Callee,
              std::vector<std::unique_ptr<ExprAST>> Args)
    : Callee(Callee), Args(std::move(Args)) {}
};
```

Tất cả đều (một cách có chủ ý) khá đơn giản: biến lưu tên biến, toán tử nhị phân lưu opcode (mã phép toán) của nó (ví dụ: '+'), còn lời gọi hàm lưu tên hàm cùng danh sách các biểu thức đối số. Một điểm hay của AST này là nó nắm bắt các đặc tính của ngôn ngữ mà không hề đề cập đến cú pháp của ngôn ngữ. Lưu ý rằng ở đây không có gì nói về độ ưu tiên của toán tử nhị phân, cấu trúc từ vựng, v.v.

Với ngôn ngữ cơ bản của chúng ta, đây là tất cả các nút biểu thức mà chúng ta sẽ định nghĩa. Vì chưa có luồng điều khiển có điều kiện nên ngôn ngữ này chưa Turing-complete (đầy đủ Turing); chúng ta sẽ khắc phục điều đó ở một phần sau. Hai thứ tiếp theo chúng ta cần là một cách để mô tả giao diện của một hàm, và một cách để mô tả chính các hàm:

```c++
/// PrototypeAST - Lớp này biểu diễn "prototype" của một hàm,
/// lưu tên hàm và tên các đối số của nó (do đó ngầm định luôn số lượng
/// đối số mà hàm nhận).
class PrototypeAST {
  std::string Name;
  std::vector<std::string> Args;

public:
  PrototypeAST(const std::string &Name, std::vector<std::string> Args)
    : Name(Name), Args(std::move(Args)) {}

  const std::string &getName() const { return Name; }
};

/// FunctionAST - Lớp này biểu diễn chính định nghĩa của một hàm.
class FunctionAST {
  std::unique_ptr<PrototypeAST> Proto;
  std::unique_ptr<ExprAST> Body;

public:
  FunctionAST(std::unique_ptr<PrototypeAST> Proto,
              std::unique_ptr<ExprAST> Body)
    : Proto(std::move(Proto)), Body(std::move(Body)) {}
};
```

Trong Kaleidoscope, kiểu của hàm chỉ được xác định bởi số lượng đối số. Vì mọi giá trị đều là số thực dấu phẩy động độ chính xác kép (double), nên không cần lưu kiểu của từng đối số ở đâu cả. Trong một ngôn ngữ thực tế và tham vọng hơn, lớp "ExprAST" có lẽ sẽ có thêm một trường kiểu.

Với bộ khung này, giờ chúng ta có thể bàn về việc phân tích biểu thức và thân hàm trong Kaleidoscope.

## Cơ bản về Parser

Giờ đã có AST để xây dựng, chúng ta cần định nghĩa mã parser để xây dựng nó. Ý tưởng ở đây là chúng ta muốn phân tích một thứ như "x+y" (được lexer trả về dưới dạng ba token) thành một AST có thể được tạo ra bằng những lời gọi như sau:

```c++
auto LHS = std::make_unique<VariableExprAST>("x");
auto RHS = std::make_unique<VariableExprAST>("y");
auto Result = std::make_unique<BinaryExprAST>('+', std::move(LHS),
                                              std::move(RHS));
```

Để làm được điều này, chúng ta sẽ bắt đầu bằng việc định nghĩa một số hàm hỗ trợ cơ bản:

```c++
/// CurTok/getNextToken - Cung cấp một bộ đệm token đơn giản. CurTok là token
/// hiện tại mà parser đang xét. getNextToken đọc token tiếp theo từ lexer
/// và cập nhật CurTok bằng kết quả đó.
static int CurTok;
static int getNextToken() {
  return CurTok = gettok();
}
```

Đoạn mã này hiện thực một bộ đệm token đơn giản bao quanh lexer. Nó cho phép chúng ta nhìn trước một token trong số những gì lexer trả về. Mọi hàm trong parser sẽ giả định rằng CurTok là token hiện tại cần được phân tích.

```c++
/// LogError* - Đây là các hàm hỗ trợ nhỏ để xử lý lỗi.
std::unique_ptr<ExprAST> LogError(const char *Str) {
  fprintf(stderr, "Error: %s\n", Str);
  return nullptr;
}
std::unique_ptr<PrototypeAST> LogErrorP(const char *Str) {
  LogError(Str);
  return nullptr;
}
```

Các hàm `LogError` là những hàm hỗ trợ đơn giản mà parser sẽ dùng để xử lý lỗi. Cơ chế phục hồi lỗi trong parser của chúng ta sẽ không phải là tốt nhất và cũng không thân thiện lắm với người dùng, nhưng đủ dùng cho hướng dẫn này. Các hàm này giúp việc xử lý lỗi trong những hàm có kiểu trả về khác nhau trở nên dễ dàng hơn: chúng luôn trả về null.

Với các hàm hỗ trợ cơ bản này, chúng ta có thể hiện thực phần đầu tiên của ngữ pháp: các literal số.

## Phân tích biểu thức cơ bản

Chúng ta bắt đầu với literal số vì chúng đơn giản nhất để xử lý. Với mỗi luật sinh (production) trong ngữ pháp, chúng ta sẽ định nghĩa một hàm để phân tích luật sinh đó. Với literal số, ta có:

```c++
/// numberexpr ::= number
static std::unique_ptr<ExprAST> ParseNumberExpr() {
  auto Result = std::make_unique<NumberExprAST>(NumVal);
  getNextToken(); // tiêu thụ số
  return std::move(Result);
}
```

Hàm này rất đơn giản: nó được kỳ vọng sẽ được gọi khi token hiện tại là token `tok_number`. Nó lấy giá trị số hiện tại, tạo một nút `NumberExprAST`, đưa lexer tiến tới token tiếp theo, và cuối cùng trả về.

Có một vài khía cạnh thú vị ở đây. Quan trọng nhất là hàm này "ăn" (tiêu thụ) toàn bộ các token tương ứng với luật sinh và trả lại bộ đệm lexer với token tiếp theo (không thuộc luật sinh ngữ pháp này) đã sẵn sàng. Đây là cách làm khá chuẩn mực đối với các parser đệ quy đi xuống. Để có một ví dụ rõ hơn, toán tử dấu ngoặc đơn được định nghĩa như sau:

```c++
/// parenexpr ::= '(' expression ')'
static std::unique_ptr<ExprAST> ParseParenExpr() {
  getNextToken(); // ăn (.
  auto V = ParseExpression();
  if (!V)
    return nullptr;

  if (CurTok != ')')
    return LogError("expected ')'");
  getNextToken(); // ăn ).
  return V;
}
```

Hàm này minh họa một số điều thú vị về parser:

1\) Nó cho thấy cách chúng ta dùng các hàm LogError. Khi được gọi, hàm này kỳ vọng token hiện tại là token '(', nhưng sau khi phân tích biểu thức con, có thể sẽ không có ')' nào đang chờ. Ví dụ, nếu người dùng gõ "(4 x" thay vì "(4)", parser cần phát ra lỗi. Vì lỗi có thể xảy ra, parser cần một cách để báo hiệu rằng lỗi đã xảy ra: trong parser của chúng ta, ta trả về null khi gặp lỗi.

2\) Một khía cạnh thú vị khác của hàm này là nó dùng đệ quy bằng cách gọi `ParseExpression` (chúng ta sẽ sớm thấy `ParseExpression` có thể gọi `ParseParenExpr`). Điều này rất mạnh mẽ vì nó cho phép xử lý các ngữ pháp đệ quy và giữ cho mỗi luật sinh thật đơn giản. Lưu ý rằng bản thân dấu ngoặc đơn không tạo ra nút AST. Dù có thể làm theo cách đó, vai trò quan trọng nhất của dấu ngoặc đơn là định hướng cho parser và nhóm các thành phần lại với nhau. Một khi parser đã xây dựng xong AST, dấu ngoặc đơn không còn cần thiết nữa.

Luật sinh đơn giản tiếp theo dùng để xử lý tham chiếu biến và lời gọi hàm:

```c++
/// identifierexpr
///   ::= identifier
///   ::= identifier '(' expression* ')'
static std::unique_ptr<ExprAST> ParseIdentifierExpr() {
  std::string IdName = IdentifierStr;

  getNextToken();  // ăn identifier.

  if (CurTok != '(') // Tham chiếu biến đơn giản.
    return std::make_unique<VariableExprAST>(IdName);

  // Lời gọi hàm.
  getNextToken();  // ăn (
  std::vector<std::unique_ptr<ExprAST>> Args;
  if (CurTok != ')') {
    while (true) {
      if (auto Arg = ParseExpression())
        Args.push_back(std::move(Arg));
      else
        return nullptr;

      if (CurTok == ')')
        break;

      if (CurTok != ',')
        return LogError("Expected ')' or ',' in argument list");
      getNextToken();
    }
  }

  // Ăn ')'.
  getNextToken();

  return std::make_unique<CallExprAST>(IdName, std::move(Args));
}
```

Hàm này theo cùng phong cách với các hàm khác. (Nó được kỳ vọng sẽ được gọi khi token hiện tại là token `tok_identifier`.) Nó cũng có đệ quy và xử lý lỗi. Một điểm thú vị là nó dùng kỹ thuật *nhìn trước* (look-ahead) để xác định identifier hiện tại là một tham chiếu biến độc lập hay là một biểu thức gọi hàm. Nó xử lý điều này bằng cách kiểm tra xem token đứng sau identifier có phải là token '(' hay không, rồi tạo nút `VariableExprAST` hoặc `CallExprAST` tương ứng.

Giờ khi đã có toàn bộ logic phân tích các biểu thức đơn giản, chúng ta có thể định nghĩa một hàm hỗ trợ để gói chúng lại vào một điểm vào duy nhất. Chúng ta gọi nhóm biểu thức này là biểu thức "primary" (biểu thức cơ sở), vì những lý do sẽ trở nên rõ ràng hơn [ở phần sau của hướng dẫn](LangImpl06.md#user-defined-unary-operators). Để phân tích một biểu thức primary bất kỳ, chúng ta cần xác định nó thuộc loại biểu thức nào:

```c++
/// primary
///   ::= identifierexpr
///   ::= numberexpr
///   ::= parenexpr
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
  }
}
```

Giờ khi đã thấy định nghĩa của hàm này, sẽ rõ ràng hơn vì sao chúng ta có thể giả định trạng thái của CurTok trong các hàm khác nhau. Hàm này dùng look-ahead để xác định loại biểu thức đang được xét, rồi phân tích nó bằng một lời gọi hàm.

Giờ khi các biểu thức cơ bản đã được xử lý, chúng ta cần xử lý biểu thức nhị phân. Chúng phức tạp hơn một chút.

## Phân tích biểu thức nhị phân

Biểu thức nhị phân khó phân tích hơn đáng kể vì chúng thường mơ hồ. Ví dụ, với chuỗi "x+y\*z", parser có thể chọn phân tích thành "(x+y)\*z" hoặc "x+(y\*z)". Theo các quy ước thông thường trong toán học, chúng ta mong đợi cách phân tích thứ hai, vì "\*" (phép nhân) có *độ ưu tiên* (precedence) cao hơn "+" (phép cộng).

Có nhiều cách để xử lý vấn đề này, nhưng một cách thanh lịch và hiệu quả là dùng [Phân tích theo độ ưu tiên toán tử](http://en.wikipedia.org/wiki/Operator-precedence_parser). Kỹ thuật phân tích này dùng độ ưu tiên của các toán tử nhị phân để định hướng đệ quy. Để bắt đầu, chúng ta cần một bảng độ ưu tiên:

```c++
/// BinopPrecedence - Lưu độ ưu tiên của từng toán tử nhị phân
/// đã được định nghĩa.
static std::map<char, int> BinopPrecedence;

/// GetTokPrecedence - Lấy độ ưu tiên của token toán tử nhị phân đang chờ xử lý.
static int GetTokPrecedence() {
  if (!isascii(CurTok))
    return -1;

  // Đảm bảo đây là một binop đã được khai báo.
  int TokPrec = BinopPrecedence[CurTok];
  if (TokPrec <= 0) return -1;
  return TokPrec;
}

int main() {
  // Cài đặt các toán tử nhị phân chuẩn.
  // 1 là độ ưu tiên thấp nhất.
  BinopPrecedence['<'] = 10;
  BinopPrecedence['+'] = 20;
  BinopPrecedence['-'] = 20;
  BinopPrecedence['*'] = 40;  // cao nhất.
  ...
}
```

Với dạng cơ bản của Kaleidoscope, chúng ta chỉ hỗ trợ 4 toán tử nhị phân (tất nhiên bạn – người đọc dũng cảm và gan dạ của chúng tôi – có thể mở rộng thêm). Hàm `GetTokPrecedence` trả về độ ưu tiên của token hiện tại, hoặc -1 nếu token đó không phải là toán tử nhị phân. Dùng map giúp dễ dàng thêm toán tử mới và làm rõ rằng thuật toán không phụ thuộc vào các toán tử cụ thể nào, nhưng cũng khá dễ để bỏ map đi và thực hiện các phép so sánh ngay trong hàm `GetTokPrecedence`. (Hoặc đơn giản là dùng một mảng có kích thước cố định.)

Với hàm hỗ trợ ở trên, giờ chúng ta có thể bắt đầu phân tích biểu thức nhị phân. Ý tưởng cơ bản của phân tích theo độ ưu tiên toán tử là chia nhỏ một biểu thức chứa các toán tử nhị phân có khả năng gây mơ hồ thành nhiều phần. Chẳng hạn, xét biểu thức "a+b+(c+d)\*e\*f+g". Phân tích theo độ ưu tiên toán tử coi đây là một dòng các biểu thức primary được phân tách bởi các toán tử nhị phân. Do đó, đầu tiên nó sẽ phân tích biểu thức primary đứng đầu là "a", sau đó sẽ thấy các cặp [+, b] [+, (c+d)] [\*, e] [\*, f] và [+, g]. Lưu ý rằng vì biểu thức trong ngoặc đơn là biểu thức primary, nên parser biểu thức nhị phân hoàn toàn không cần bận tâm đến các biểu thức con lồng nhau như (c+d).

Để bắt đầu, một biểu thức là một biểu thức primary, có thể theo sau bởi một chuỗi các cặp [binop,primaryexpr]:

```c++
/// expression
///   ::= primary binoprhs
///
static std::unique_ptr<ExprAST> ParseExpression() {
  auto LHS = ParsePrimary();
  if (!LHS)
    return nullptr;

  return ParseBinOpRHS(0, std::move(LHS));
}
```

`ParseBinOpRHS` là hàm phân tích chuỗi các cặp này cho chúng ta. Nó nhận vào một độ ưu tiên và một con trỏ tới biểu thức của phần đã được phân tích cho đến lúc đó. Lưu ý rằng "x" là một biểu thức hoàn toàn hợp lệ: do đó, "binoprhs" được phép rỗng, khi đó hàm sẽ trả về chính biểu thức được truyền vào. Trong ví dụ ở trên, mã truyền biểu thức "a" vào `ParseBinOpRHS` và token hiện tại là "+".

Giá trị độ ưu tiên được truyền vào `ParseBinOpRHS` cho biết *độ ưu tiên toán tử tối thiểu* mà hàm được phép ăn. Ví dụ, nếu dòng cặp hiện tại là [+, x] và `ParseBinOpRHS` được truyền độ ưu tiên 40, nó sẽ không tiêu thụ token nào (vì độ ưu tiên của '+' chỉ là 20). Với điều này trong đầu, `ParseBinOpRHS` bắt đầu như sau:

```c++
/// binoprhs
///   ::= ('+' primary)*
static std::unique_ptr<ExprAST> ParseBinOpRHS(int ExprPrec,
                                              std::unique_ptr<ExprAST> LHS) {
  // Nếu đây là một binop, tìm độ ưu tiên của nó.
  while (true) {
    int TokPrec = GetTokPrecedence();

    // Nếu đây là binop liên kết ít nhất là chặt bằng binop hiện tại,
    // thì tiêu thụ nó, nếu không thì ta đã xong.
    if (TokPrec < ExprPrec)
      return LHS;
```

Đoạn mã này lấy độ ưu tiên của token hiện tại và kiểm tra xem nó có quá thấp hay không. Vì chúng ta đã định nghĩa các token không hợp lệ có độ ưu tiên là -1, phép kiểm tra này ngầm hiểu rằng dòng cặp sẽ kết thúc khi dòng token không còn toán tử nhị phân nào. Nếu phép kiểm tra này vượt qua, ta biết rằng token đó là một toán tử nhị phân và nó sẽ được đưa vào biểu thức này:

```c++
// Được rồi, ta biết đây là một binop.
int BinOp = CurTok;
getNextToken();  // ăn binop

// Phân tích biểu thức primary đứng sau toán tử nhị phân.
auto RHS = ParsePrimary();
if (!RHS)
  return nullptr;
```

Như vậy, đoạn mã này ăn (và ghi nhớ) toán tử nhị phân rồi phân tích biểu thức primary theo sau. Việc này tạo nên một cặp hoàn chỉnh, mà cặp đầu tiên trong ví dụ đang xét là [+, b].

Giờ khi đã phân tích vế trái của biểu thức và một cặp của chuỗi RHS, chúng ta phải quyết định biểu thức kết hợp theo hướng nào. Cụ thể, chúng ta có thể có "(a+b) binop unparsed" hoặc "a + (b binop unparsed)". Để xác định điều này, ta nhìn trước "binop" để biết độ ưu tiên của nó và so sánh với độ ưu tiên của BinOp (trong trường hợp này là '+'):

```c++
// Nếu BinOp liên kết với RHS lỏng hơn toán tử đứng sau RHS, hãy để
// toán tử đang chờ lấy RHS làm LHS của nó.
int NextPrec = GetTokPrecedence();
if (TokPrec < NextPrec) {
```

Nếu độ ưu tiên của binop nằm bên phải "RHS" thấp hơn hoặc bằng độ ưu tiên của toán tử hiện tại, ta biết rằng dấu ngoặc sẽ kết hợp theo dạng "(a+b) binop ...". Trong ví dụ của chúng ta, toán tử hiện tại là "+" và toán tử tiếp theo cũng là "+", nên ta biết chúng có cùng độ ưu tiên. Trong trường hợp này, ta sẽ tạo nút AST cho "a+b", rồi tiếp tục phân tích:

```c++
      ... phần thân if được lược bỏ ...
    }

    // Gộp LHS/RHS.
    LHS = std::make_unique<BinaryExprAST>(BinOp, std::move(LHS),
                                           std::move(RHS));
  }  // quay lại đầu vòng lặp while.
}
```

Trong ví dụ ở trên, việc này sẽ biến "a+b+" thành "(a+b)" và thực hiện lần lặp tiếp theo của vòng lặp, với "+" là token hiện tại. Đoạn mã trên sẽ ăn, ghi nhớ và phân tích "(c+d)" như một biểu thức primary, khiến cặp hiện tại trở thành [+, (c+d)]. Sau đó, nó sẽ đánh giá điều kiện 'if' ở trên với "\*" là binop nằm bên phải biểu thức primary. Trong trường hợp này, độ ưu tiên của "\*" cao hơn độ ưu tiên của "+", nên chương trình sẽ đi vào nhánh if.

Câu hỏi then chốt còn lại ở đây là "làm thế nào để nhánh if phân tích trọn vẹn vế phải?" Cụ thể, để xây dựng AST chính xác cho ví dụ của chúng ta, nó cần lấy toàn bộ "(c+d)\*e\*f" làm biến biểu thức RHS. Mã để làm điều này đơn giản đến bất ngờ (mã từ hai khối ở trên được lặp lại để làm ngữ cảnh):

```c++
    // Nếu BinOp liên kết với RHS lỏng hơn toán tử đứng sau RHS, hãy để
    // toán tử đang chờ lấy RHS làm LHS của nó.
    int NextPrec = GetTokPrecedence();
    if (TokPrec < NextPrec) {
      RHS = ParseBinOpRHS(TokPrec+1, std::move(RHS));
      if (!RHS)
        return nullptr;
    }
    // Gộp LHS/RHS.
    LHS = std::make_unique<BinaryExprAST>(BinOp, std::move(LHS),
                                           std::move(RHS));
  }  // quay lại đầu vòng lặp while.
}
```

Tại thời điểm này, ta biết rằng toán tử nhị phân nằm bên phải biểu thức primary có độ ưu tiên cao hơn binop mà ta đang phân tích. Do đó, ta biết rằng mọi chuỗi cặp có các toán tử đều có độ ưu tiên cao hơn "+" nên được phân tích cùng nhau và trả về dưới dạng "RHS". Để làm điều này, ta gọi đệ quy hàm `ParseBinOpRHS` với "TokPrec+1" là độ ưu tiên tối thiểu cần có để nó tiếp tục. Trong ví dụ ở trên, điều này khiến hàm trả về nút AST cho "(c+d)\*e\*f" làm RHS, rồi nút này được đặt làm RHS của biểu thức '+'.

Cuối cùng, ở lần lặp tiếp theo của vòng lặp while, phần "+g" được phân tích và thêm vào AST. Chỉ với một chút mã này (14 dòng không tầm thường), chúng ta đã xử lý chính xác việc phân tích biểu thức nhị phân tổng quát một cách rất thanh lịch. Đây là một chuyến tham quan chớp nhoáng qua đoạn mã này, và nó có phần tinh tế. Tôi khuyên bạn nên chạy thử nó với vài ví dụ khó để thấy nó hoạt động như thế nào.

Đến đây, phần xử lý biểu thức đã hoàn tất. Lúc này, ta có thể đưa cho parser một dòng token bất kỳ và xây dựng một biểu thức từ đó, dừng lại tại token đầu tiên không thuộc biểu thức. Tiếp theo, chúng ta cần xử lý định nghĩa hàm, v.v.

## Phân tích phần còn lại

Thứ còn thiếu tiếp theo là xử lý prototype của hàm. Trong Kaleidoscope, prototype được dùng cho cả khai báo hàm 'extern' lẫn định nghĩa thân hàm. Mã để làm việc này khá đơn giản và không có gì thú vị lắm (một khi bạn đã vượt qua được phần biểu thức):

```c++
/// prototype
///   ::= id '(' id* ')'
static std::unique_ptr<PrototypeAST> ParsePrototype() {
  if (CurTok != tok_identifier)
    return LogErrorP("Expected function name in prototype");

  std::string FnName = IdentifierStr;
  getNextToken();

  if (CurTok != '(')
    return LogErrorP("Expected '(' in prototype");

  // Đọc danh sách tên các đối số.
  std::vector<std::string> ArgNames;
  while (getNextToken() == tok_identifier)
    ArgNames.push_back(IdentifierStr);
  if (CurTok != ')')
    return LogErrorP("Expected ')' in prototype");

  // thành công.
  getNextToken();  // ăn ')'.

  return std::make_unique<PrototypeAST>(FnName, std::move(ArgNames));
}
```

Với phần này, một định nghĩa hàm trở nên rất đơn giản, chỉ gồm một prototype cộng với một biểu thức để hiện thực phần thân:

```c++
/// definition ::= 'def' prototype expression
static std::unique_ptr<FunctionAST> ParseDefinition() {
  getNextToken();  // ăn def.
  auto Proto = ParsePrototype();
  if (!Proto) return nullptr;

  if (auto E = ParseExpression())
    return std::make_unique<FunctionAST>(std::move(Proto), std::move(E));
  return nullptr;
}
```

Ngoài ra, chúng ta hỗ trợ 'extern' để khai báo các hàm như 'sin' và 'cos', cũng như hỗ trợ khai báo trước (forward declaration) các hàm do người dùng định nghĩa. Các 'extern' này chỉ đơn thuần là prototype không có thân hàm:

```c++
/// external ::= 'extern' prototype
static std::unique_ptr<PrototypeAST> ParseExtern() {
  getNextToken();  // ăn extern.
  return ParsePrototype();
}
```

Cuối cùng, chúng ta cũng cho phép người dùng gõ các biểu thức cấp cao nhất (top-level) bất kỳ và đánh giá chúng ngay lập tức. Chúng ta sẽ xử lý việc này bằng cách định nghĩa các hàm ẩn danh không đối số (nullary) cho chúng:

```c++
/// toplevelexpr ::= expression
static std::unique_ptr<FunctionAST> ParseTopLevelExpr() {
  if (auto E = ParseExpression()) {
    // Tạo một prototype ẩn danh.
    auto Proto = std::make_unique<PrototypeAST>("__anon_expr", std::vector<std::string>());
    return std::make_unique<FunctionAST>(std::move(Proto), std::move(E));
  }
  return nullptr;
}
```

Giờ khi đã có đủ mọi mảnh ghép, hãy xây dựng một driver nhỏ để thực sự *chạy* đoạn mã mà chúng ta đã xây dựng!

## Driver

Driver ở đây chỉ đơn giản gọi tất cả các phần phân tích cú pháp thông qua một vòng lặp điều phối cấp cao nhất. Không có gì quá thú vị ở đây, nên tôi sẽ chỉ đưa vào vòng lặp cấp cao nhất. Xem [bên dưới](#full-code-listing) để có mã đầy đủ trong phần "Top-Level Parsing".

```c++
/// top ::= definition | external | expression | ';'
static void MainLoop() {
  while (true) {
    fprintf(stderr, "ready> ");
    switch (CurTok) {
    case tok_eof:
      return;
    case ';': // bỏ qua dấu chấm phẩy ở cấp cao nhất.
      getNextToken();
      break;
    case tok_def:
      HandleDefinition();
      break;
    case tok_extern:
      HandleExtern();
      break;
    default:
      HandleTopLevelExpression();
      break;
    }
  }
}
```

Phần thú vị nhất ở đây là chúng ta bỏ qua các dấu chấm phẩy ở cấp cao nhất. Bạn hỏi tại sao ư? Lý do cơ bản là nếu bạn gõ "4 + 5" tại dòng lệnh, parser không biết liệu đó đã là phần kết thúc của những gì bạn sẽ gõ hay chưa. Ví dụ, ở dòng tiếp theo bạn có thể gõ "def foo...", khi đó 4+5 là phần kết thúc của một biểu thức cấp cao nhất. Hoặc bạn cũng có thể gõ "\* 6", và như vậy biểu thức sẽ được tiếp tục. Việc có dấu chấm phẩy ở cấp cao nhất cho phép bạn gõ "4+5;", và parser sẽ biết bạn đã gõ xong.

## Kết luận

Chỉ với chưa đầy 400 dòng mã có chú thích (240 dòng mã không tính chú thích và dòng trống), chúng ta đã định nghĩa đầy đủ ngôn ngữ tối giản của mình, bao gồm lexer, parser và bộ xây dựng AST. Khi đã hoàn tất, chương trình thực thi sẽ kiểm tra mã Kaleidoscope và cho chúng ta biết nếu mã đó sai ngữ pháp. Ví dụ, đây là một phiên tương tác mẫu:

```bash
$ ./a.out
ready> def foo(x y) x+foo(y, 4.0);
Parsed a function definition.
ready> def foo(x y) x+y y;
Parsed a function definition.
Parsed a top-level expr
ready> def foo(x y) x+y );
Parsed a function definition.
Error: unknown token when expecting an expression
ready> extern sin(a);
ready> Parsed an extern
ready> ^D
$
```

Vẫn còn rất nhiều chỗ để mở rộng ở đây. Bạn có thể định nghĩa các nút AST mới, mở rộng ngôn ngữ theo nhiều cách, v.v. Trong [phần tiếp theo](LangImpl03.md), chúng ta sẽ mô tả cách sinh Biểu diễn trung gian (Intermediate Representation – IR) của LLVM từ AST.

<a id="full-code-listing"></a>

## Mã nguồn đầy đủ

Dưới đây là toàn bộ mã nguồn cho ví dụ xuyên suốt của chúng ta.

```bash
# Biên dịch
clang++ -g -O3 toy.cpp
# Chạy
./a.out
```

Đây là mã nguồn:

```{literalinclude} ../../../examples/Kaleidoscope/Chapter2/toy.cpp
:language: c++
```

[Tiếp theo: Hiện thực sinh mã sang LLVM IR](LangImpl03.md)
