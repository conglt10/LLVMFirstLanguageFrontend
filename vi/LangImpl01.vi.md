# Kaleidoscope: Giới thiệu Kaleidoscope và Lexer

## Ngôn ngữ Kaleidoscope

Hướng dẫn này được minh họa bằng một ngôn ngữ đơn giản (toy language) có tên là "[Kaleidoscope](http://en.wikipedia.org/wiki/Kaleidoscope)" (bắt nguồn từ các từ mang nghĩa "đẹp, hình dạng và góc nhìn"). Kaleidoscope là một ngôn ngữ thủ tục (procedural language) cho phép bạn định nghĩa hàm, sử dụng câu lệnh điều kiện, các phép toán, v.v. Trong suốt hướng dẫn, chúng ta sẽ mở rộng Kaleidoscope để hỗ trợ cấu trúc if/then/else, vòng lặp for, toán tử do người dùng định nghĩa, biên dịch JIT với một giao diện dòng lệnh đơn giản, thông tin gỡ lỗi (debug info), v.v.

Chúng ta muốn giữ mọi thứ đơn giản, vì vậy kiểu dữ liệu duy nhất trong Kaleidoscope là kiểu số thực dấu phẩy động 64-bit (hay còn gọi là 'double' theo cách nói của C). Do đó, mọi giá trị đều ngầm định là số thực độ chính xác kép và ngôn ngữ không yêu cầu khai báo kiểu. Điều này mang lại cho ngôn ngữ một cú pháp rất gọn gàng và đơn giản. Ví dụ, đoạn mã đơn giản sau đây tính [các số Fibonacci:](http://en.wikipedia.org/wiki/Fibonacci_number)

```
# Tính số fibonacci thứ x.
def fib(x)
  if x < 3 then
    1
  else
    fib(x-1)+fib(x-2)

# Biểu thức này sẽ tính số thứ 40.
fib(40)
```

Chúng ta cũng cho phép Kaleidoscope gọi các hàm trong thư viện chuẩn - LLVM JIT giúp việc này trở nên vô cùng dễ dàng. Điều này có nghĩa là bạn có thể dùng từ khóa 'extern' để khai báo một hàm trước khi sử dụng nó (điều này cũng hữu ích cho các hàm đệ quy lẫn nhau). Ví dụ:

```
extern sin(arg);
extern cos(arg);
extern atan2(arg1 arg2);

atan2(sin(.4), cos(42))
```

Một ví dụ thú vị hơn có trong Chương 6, nơi chúng ta viết một ứng dụng Kaleidoscope nhỏ [hiển thị tập Mandelbrot](LangImpl06.md#kicking-the-tires) ở nhiều mức độ phóng đại khác nhau.

Hãy cùng đi sâu vào việc hiện thực ngôn ngữ này!

## Lexer

Khi hiện thực một ngôn ngữ, điều đầu tiên cần có là khả năng xử lý một tệp văn bản và nhận biết nội dung của nó. Cách truyền thống để làm việc này là sử dụng một "[lexer](http://en.wikipedia.org/wiki/Lexical_analysis)" (bộ phân tích từ vựng, hay còn gọi là 'scanner') để chia nhỏ đầu vào thành các "token". Mỗi token mà lexer trả về bao gồm một mã token và có thể kèm theo một số siêu dữ liệu (metadata) (ví dụ: giá trị số của một con số). Trước tiên, chúng ta định nghĩa các khả năng có thể xảy ra:

```c++
// Lexer trả về token [0-255] nếu đó là một ký tự không xác định, ngược lại
// trả về một trong các giá trị sau cho những thứ đã biết.
enum Token {
  tok_eof = -1,

  // lệnh
  tok_def = -2,
  tok_extern = -3,

  // cơ bản
  tok_identifier = -4,
  tok_number = -5,
};

static std::string IdentifierStr; // Được gán giá trị nếu là tok_identifier
static double NumVal;             // Được gán giá trị nếu là tok_number
```

Mỗi token mà lexer của chúng ta trả về sẽ hoặc là một trong các giá trị của enum Token, hoặc là một ký tự 'không xác định' như '+', được trả về dưới dạng giá trị ASCII của nó. Nếu token hiện tại là một định danh (identifier), biến toàn cục `IdentifierStr` sẽ lưu tên của định danh đó. Nếu token hiện tại là một hằng số (numeric literal) (như 1.0), `NumVal` sẽ lưu giá trị của nó. Chúng ta dùng biến toàn cục cho đơn giản, nhưng đây không phải là lựa chọn tốt nhất cho việc hiện thực một ngôn ngữ thực tế :).

Phần hiện thực thực sự của lexer là một hàm duy nhất có tên `gettok`. Hàm `gettok` được gọi để trả về token tiếp theo từ đầu vào chuẩn (standard input). Định nghĩa của nó bắt đầu như sau:

```c++
/// gettok - Trả về token tiếp theo từ đầu vào chuẩn.
static int gettok() {
  static int LastChar = ' ';

  // Bỏ qua mọi khoảng trắng.
  while (isspace(LastChar))
    LastChar = getchar();
```

`gettok` hoạt động bằng cách gọi hàm C `getchar()` để đọc từng ký tự một từ đầu vào chuẩn. Nó "tiêu thụ" các ký tự khi nhận diện được chúng và lưu ký tự cuối cùng đã đọc nhưng chưa được xử lý vào LastChar. Việc đầu tiên nó phải làm là bỏ qua khoảng trắng giữa các token. Điều này được thực hiện bằng vòng lặp ở trên.

Việc tiếp theo `gettok` cần làm là nhận diện các định danh và các từ khóa cụ thể như "def". Kaleidoscope thực hiện điều này bằng vòng lặp đơn giản sau:

```c++
if (isalpha(LastChar)) { // định danh: [a-zA-Z][a-zA-Z0-9]*
  IdentifierStr = LastChar;
  while (isalnum((LastChar = getchar())))
    IdentifierStr += LastChar;

  if (IdentifierStr == "def")
    return tok_def;
  if (IdentifierStr == "extern")
    return tok_extern;
  return tok_identifier;
}
```

Lưu ý rằng đoạn mã này gán giá trị cho biến toàn cục '`IdentifierStr`' mỗi khi nó phân tích từ vựng một định danh. Ngoài ra, vì các từ khóa của ngôn ngữ được so khớp bởi cùng một vòng lặp, chúng ta xử lý chúng ngay tại đây. Các giá trị số được xử lý tương tự:

```c++
if (isdigit(LastChar) || LastChar == '.') {   // Số: [0-9.]+
  std::string NumStr;
  do {
    NumStr += LastChar;
    LastChar = getchar();
  } while (isdigit(LastChar) || LastChar == '.');

  NumVal = strtod(NumStr.c_str(), 0);
  return tok_number;
}
```

Tất cả đều là mã xử lý đầu vào khá đơn giản. Khi đọc một giá trị số từ đầu vào, chúng ta dùng hàm C `strtod` để chuyển nó thành giá trị số và lưu vào `NumVal`. Lưu ý rằng đoạn mã này chưa kiểm tra lỗi đầy đủ: nó sẽ đọc sai "1.23.45.67" và xử lý như thể bạn đã nhập "1.23". Bạn cứ thoải mái mở rộng nó! Tiếp theo, chúng ta xử lý chú thích (comment):

```c++
if (LastChar == '#') {
  // Chú thích kéo dài đến cuối dòng.
  do
    LastChar = getchar();
  while (LastChar != EOF && LastChar != '\n' && LastChar != '\r');

  if (LastChar != EOF)
    return gettok();
}
```

Chúng ta xử lý chú thích bằng cách bỏ qua cho đến cuối dòng rồi trả về token tiếp theo. Cuối cùng, nếu đầu vào không khớp với bất kỳ trường hợp nào ở trên, thì nó hoặc là một ký tự toán tử như '+', hoặc là cuối tệp. Những trường hợp này được xử lý bằng đoạn mã sau:

```c++
  // Kiểm tra cuối tệp. Không tiêu thụ EOF.
  if (LastChar == EOF)
    return tok_eof;

  // Ngược lại, chỉ cần trả về ký tự dưới dạng giá trị ascii của nó.
  int ThisChar = LastChar;
  LastChar = getchar();
  return ThisChar;
}
```

Với những phần trên, chúng ta đã có một lexer hoàn chỉnh cho ngôn ngữ Kaleidoscope cơ bản ([toàn bộ mã nguồn](LangImpl02.md#full-code-listing) của Lexer có trong [chương tiếp theo](LangImpl02.md) của hướng dẫn). Tiếp theo, chúng ta sẽ [xây dựng một parser đơn giản sử dụng lexer này để tạo Cây cú pháp trừu tượng (Abstract Syntax Tree)](LangImpl02.md). Khi đã có nó, chúng ta sẽ thêm một chương trình điều khiển (driver) để bạn có thể sử dụng lexer và parser cùng nhau.

[Tiếp theo: Hiện thực Parser và AST](LangImpl02.vi.md)
